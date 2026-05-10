# GitOps CD -- GitHub Actions + ArgoCD Integration -- The Shipping Department, the Customs Checkpoint, and the Delivery Dispatcher

> The May 7 doc closed out the **factory floor**: how a `release.yml` builds, scans, signs, attests and pushes a container image to ECR with OIDC, ending at a single line of output -- `outputs.digest = sha256:abc...`. Today is the day that line travels. The factory has produced a sealed container; what happens between "the loaf is in the bonded warehouse" and "the diner is eating it in production" is the entire discipline of GitOps CD: the **shipping manifest book** (the config repo) gets a new entry, the **customs officer** (GitHub Environments) checks the paperwork, the **delivery dispatcher** (ArgoCD) drives the container to the right dock, and the dispatcher **reports back** so the manifest book closes the loop. Any pipeline that lets CI drive `kubectl apply` directly is a factory truck running deliveries itself -- it works once, on a quiet Sunday, and burns the building down by the third Tuesday.
>
> **The core analogy for today: a factory's shipping department, a customs checkpoint, and a delivery dispatcher.** Walk it through. The factory floor (last week's container CI) produces a sealed container with a tamper-evident seal. The container moves to the **bonded warehouse** -- the OCI registry, ECR or GHCR. A clerk in the shipping department then writes a single line in the **shipping manifest book** -- the config repo, a Git repository whose YAML files are the legal record of "container `sha256:abc...` is destined for dock `dev` on date `2026-05-10`." That manifest book is the source of truth; nothing leaves the warehouse except by what the book says. **GitHub Environments is the customs checkpoint** -- the manifest can be written instantly for the dev dock, but for the prod dock it needs a stamp from a customs officer (required reviewers), an inspection delay (wait timer), and the right import lane (deployment branch policy). **ArgoCD is the delivery dispatcher** -- it stands at every dock, reads the manifest book continuously, and drives whichever container the book points to into that dock. When the container arrives, ArgoCD radios back: "container delivered, dock dev, healthy" -- a status update written onto the original commit in the manifest book, which appears as a green check next to the developer's PR. **Image Updater is an automated manifest-clerk** -- a robot inventory-watcher that adds new entries to the manifest book whenever a fresh container shows up at the warehouse, but writes in pen, not pencil (every change is a real Git commit, not a hidden in-cluster override). **Kargo is a freight broker** -- when a single shipment has to clear three borders (dev -> stg -> prod) with handoffs, sign-offs, and SLO checks at each border, you stop hand-rolling the customs forms and hire a broker who runs the whole pipeline.
>
> Three corollaries to bolt on. **(1) The manifest book has one author per pen.** The single most common failure mode in GitOps CD is having three different automations all editing the same `values.yaml` -- CI direct-push, Image Updater write-back, and a hand-PR'd promotion all racing to bump `image.tag`. Pick one mechanism per repo, document it on the README, and treat the others as anti-patterns for that codebase. **(2) The customs stamp is on the *environment*, not the *branch*.** Yesterday's branch-scoped OIDC `sub` (`repo:org/repo:ref:refs/heads/main`) is fine for "build the image" but wrong for "deploy to prod" -- environment-scoped (`repo:org/repo:environment:production`) composes with required reviewers, wait timers, and branch policies, so AWS can only mint creds *after GitHub has already enforced human approval*. The two-line difference in your trust policy is the difference between "any push to main can take down prod" and "no production credential is ever issued without a stamp." **(3) The dispatcher reports back, or the loop never closes.** A CI workflow that exits green when the PR merges is lying -- it has only confirmed "the manifest entry was written," not "the container arrived." ArgoCD Notifications is the radio system that tells the manifest book *the truth* (deployed, healthy, sync-failed, drifting). Without it, the green check on a PR is a vanity metric and your incident postmortems will start with "the deploy looked successful in CI but..."

---

**Date**: 2026-05-10
**Topic Area**: cicd
**Difficulty**: Advanced

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| The CI/CD handoff principle | Factory ends at warehouse, dispatcher takes over | CI publishes signed image + commits digest to Git; ArgoCD reconciles -- CI never `kubectl apply`s |
| Config repo / GitOps repo | The shipping manifest book | Declarative source of truth -- every line a Git-tracked YAML statement of intent |
| App repo | The factory blueprint room | Source code, Dockerfile, chart templates -- the things engineers edit by hand |
| OCI registry | The bonded warehouse | Where signed/attested image bytes live -- digest is the cryptographic identity |
| ArgoCD Application | The dispatcher's standing order at one dock | "Take whatever the manifest says and put it in this cluster/namespace" |
| GitHub Environment | A customs checkpoint | Required reviewers + wait timer + branch policy + env-scoped secrets |
| Required reviewers | Customs officers (max 6) | Up to six users/teams; one approval suffices; secrets withheld until approved |
| Wait timer | Inspection delay (max 30 days) | Composes with reviewers; not billable while waiting |
| Deployment branch policy | Approved import lanes | Restricts which branches can deploy to this environment |
| Custom protection rule | Third-party gate (ServiceNow, Datadog) | GitHub App receives webhook, calls back approve/reject |
| Environment-scoped secret | Locker only the right shift can open | Available only when job declares `environment:` AND passes protection rules |
| OIDC `sub: ...:environment:X` | "Customs-stamped courier badge" | Composes IAM trust policy with GitHub Environment gates |
| `environment:` key on a job | "Show your badge at this checkpoint" | Creates a Deployment, gates on rules, exposes env-scoped secrets |
| Image-update strategy 1: CI commits | Factory courier walks to manifest book | `peter-evans/create-pull-request` or direct push -- highest audit fidelity |
| Image-update strategy 2: Image Updater | Robot manifest-clerk | ArgoCD watches registry, writes back to Git or live Application |
| Image-update strategy 3: Kargo / Promoter | Freight broker | Stage CRD orchestrates multi-leg promotion with verification |
| `argocd` write-back (Image Updater) | Robot scribbles on the dispatcher's clipboard | DON'T -- override lives in cluster state, drifts vs Git, lost on App recreate |
| `git` write-back (Image Updater) | Robot writes in the manifest book | Production-correct -- real commit by `argocd-image-updater <noreply@argoproj.io>` |
| `helm.image-tag` annotation | "Where to write on this spice card" | Dotted path into values.yaml -- silent no-op if mismatched |
| `update-strategy: digest` | "Watch the same tag, capture the SHA" | Pins a mutable tag like `main` and updates by digest -- pairs with main-branch CI |
| `update-strategy: semver` | "Only newer v1.x" | The only safe strategy for `vX.Y.Z` releases |
| Two-repo (app + config) | Engineering vault + dispatcher's binder | Production default; permission boundaries; CI cross-writes |
| Single-repo (mono) | One folder for everything | Smaller teams; single audit timeline; weaker permission boundaries |
| Folder-per-environment | One section per dock in the manifest book | The modern default (Kapelonis): `envs/dev/`, `envs/stg/`, `envs/prod/` |
| Branch-per-environment | Separate manifest per dock | Anti-pattern -- merge nightmares within 6 months |
| Repo-per-environment | Separate book per dock | Anti-pattern -- maximum drift, no diffability |
| GITHUB_TOKEN | The factory's house badge | Cannot trigger downstream workflows -- recursion fuse |
| PAT (fine-grained) | A real human's credential | Triggers workflows; tied to a person; rotates poorly |
| GitHub App | A bonded service contractor | Triggers workflows; per-repo install; auto-rotates; the production-correct choice |
| `[skip ci]` | "Don't ring the doorbell on this commit" | Convention from GitLab/Circle. **GHA does NOT honor it natively** -- use `paths-ignore` or bot-author guards instead |
| Auto-promote dev | "Dev dock auto-receives every shipment" | Push-to-main = direct-push to `dev/values.yaml` = ArgoCD syncs dev |
| PR-promote stg/prod | "Higher docks need a signed transfer slip" | Promotion is a PR against `stg/values.yaml` -- human review required |
| Tag-driven prod | "Releases on the gold-stamped manifest only" | `git tag v1.2.3` triggers a workflow that opens a prod-promotion PR |
| `workflow_dispatch` w/ env input | "Manual override -- pick the dock" | Choice input `environment:` (dev/stg/prod) routes via `environment:` key |
| ArgoCD AppProject | Dispatcher's per-tenant rule book | Restricts which sources/destinations/clusters this batch of Apps can target |
| Kargo Stage | Freight broker's leg of the journey | One CRD per env; `requestedFreight` + `verification` + promotion steps |
| ArgoCD Notifications | The dispatcher's radio | Built-in controller; ~25 destinations; GitHub via App or PAT |
| `on-deployed` trigger | "Container delivered" radio call | Fires on Application healthy; bind to commit-status / PR-comment template |
| `on-health-degraded` trigger | "Dock is on fire" radio call | Fires when reconciled Application becomes Degraded |
| Commit status (notifications) | Green/yellow/red dot on the manifest entry | `POST /repos/{owner}/{repo}/statuses/{sha}` -- shows under PR Checks |
| GitHub Deployments API | The dock's arrival log | `POST /deployments` populates Repo > Environments sidebar |
| Recursion / infinite loop trap | Bot writes manifest, manifest triggers bot | Fix: `paths-ignore` on values files + bot-author `if:` guard. (`[skip ci]` does nothing on GHA.) |
| Drift trap (`argocd` write-back) | Two clerks scribbling over each other | Human edits values.yaml + Image Updater overrides via API -> conflict |
| Branch tracking trap | Robot writes to wrong notebook | `git-branch` annotation must equal Application's `targetRevision` |
| `helm.image-name` mismatch | Pen scribbles in the wrong column | Silent no-op; digest never bumps; debug with `argocd-image-updater test` |
| Self-approval foot-gun | Customs officer stamps own paperwork | Required reviewers can self-approve unless "Prevent self-review" enabled |
| Plan caveat | "Customs only operates at airport branches" | Required reviewers + wait timers private-repo only on Pro/Team+ |

---

## The Whole Picture in One Diagram

```
THE FACTORY -> WAREHOUSE -> CUSTOMS -> DISPATCHER -> RADIO LOOP
==============================================================================

   APP REPO  (factory blueprints)              CONFIG REPO (manifest book)
   +------------------+                        +----------------------------+
   | src/             |                        | apps/                      |
   | Dockerfile       |                        |   myapp/                   |
   | charts/myapp/    |                        |     base/values.yaml       |
   | .github/         |                        |     envs/                  |
   |   workflows/     |                        |       dev/values.yaml      |
   |     release.yml  |    1. PR/release CI    |       stg/values.yaml      |
   +--------+---------+        builds image    |       prod/values.yaml     |
            |                                  +-------------+--------------+
            |  docker push                                   ^
            v                                                | 4. ArgoCD watches
   +------------------+   2. CI commits digest               |    config repo
   |  OCI Registry    |      to envs/dev/values.yaml         |    on a poll cycle
   |  (ECR / GHCR)    |      via:                            |    (default 3 min)
   |                  |        a. peter-evans PR             |
   |  myapp@sha256... |        b. direct push (dev only)     |
   |  + cosign sig    |        c. Image Updater write-back   |
   |  + SBOM/provn    |                                      |
   +-----+------------+                                      |
         |                                                   |
         | watched by                  3. PR opened          |
         |                             promote stg/prod      |
         |                             (env protection rules |
         |                              gate the merge)      |
         v                                                   |
   +------------------+                                      |
   | Image Updater    |---commit/PR------------------------->|
   | (alt to CI write)|                                      |
   +------------------+                                      |
                                                             v
                                       +----------------------------+
                                       |  ArgoCD Application(s)     |
                                       |   per-environment          |
                                       |   AppProject scoped        |
                                       +-------------+--------------+
                                                     |
                                       helm template + apply
                                                     |
                                                     v
                                       +----------------------------+
                                       |  Cluster (per env)         |
                                       |   Deployment / Rollout     |
                                       +-------------+--------------+
                                                     |
                                                     | health/sync events
                                                     v
                                       +----------------------------+
                                       |  ArgoCD Notifications      |
                                       |   on-deployed              |
                                       |   on-health-degraded       |
                                       |   on-sync-failed           |
                                       +-------------+--------------+
                                                     |
                                                     | GitHub App / PAT
   +------------------+                              v
   |  GitHub PR       |<--- 5. commit status, deployments API
   |  /Environments   |     PR comment "Deployed to dev: healthy ✅"
   |  sidebar         |
   +------------------+

CARDINAL RULE: each numbered arrow has one owner and one auth mechanism.
              CI never writes to the cluster. ArgoCD never writes to source.
              The only thing flowing back is *status*.
```

---

## Part 1: The CI/CD Handoff Principle -- Why CI Must Never `kubectl apply`

The single load-bearing decision in GitOps CD is the **handoff contract**: where does CI end, where does CD begin, and what crosses that line? The wrong answer (CI deploys directly) is a reasonable-sounding shortcut that quietly breaks five separate things at once. Knowing which five is the whole point of this section.

### The contract in one sentence

**CI's last step is `git commit && git push` against the config repo. CD's first step is "ArgoCD detects the commit and reconciles the cluster."** That `git push` is the API surface between the two systems. Everything else -- the Helm template, the kubectl apply, the rollout health check -- belongs to ArgoCD and runs from inside the cluster.

This is not stylistic. It's the literal text of the OpenGitOps spec's third principle: **Pulled Automatically.** The cluster pulls its desired state from Git; nothing pushes state into the cluster from outside. A CI job that runs `kubectl apply` is by definition violating principle 3, because *something outside the cluster* is pushing state in. There is no clever YAML that lets you have "GitOps-style CI deploys"; the words contradict the spec.

### The five-axis comparison: CI-deploys vs ArgoCD-reconciles

(The April 19 doc walked through this. Today's contribution: connecting each axis to a CI/CD-handoff failure mode you'd actually see during incident review.)

| Axis | CI does `kubectl apply` (push) | CI commits to Git, ArgoCD reconciles (pull) |
|---|---|---|
| **Credential blast radius** | CI workflow holds long-lived kubeconfig or assumes a cluster-write IAM role. A compromised PR can invoke `kubectl delete --all` against prod. **Sharpening: OIDC eliminates *stored* credentials, not *materialized* ones** -- every workflow run still holds cluster-admin in-process for the duration. A compromised action, a malicious transitive dep, a typosquatted package -- anything that runs in CI gets full cluster write, OIDC or not. | CI holds *registry-write* + *config-repo-write* creds only -- no cluster privileges. Compromise blast radius capped at "can mutate Git, can mutate ECR" -- both reversible via `git revert` and tag-overwrite respectively. The cluster *pulls* from Git; CI is never materialized with cluster auth at all. Surface area isn't reduced -- it's eliminated. |
| **Audit trail granularity** | "Job X ran in CI on date Y" -- the *artifact* of the deploy is a CI run record, which is provider-specific, exportable, and lives in CI history (and can be deleted). | "Commit `abc1234` by `dev@org.com` on `main` of config repo at `T+0:00`, deployed by ArgoCD at `T+0:03`" -- artifact is a Git commit + ArgoCD sync record, both immutable, both diffable, both replayable. |
| **Drift handling** | None. After CI exits, the cluster's state can drift (a kubectl-edit, an HPA spec change, a sidecar injector mutating a Deployment). CI has no way to detect or correct this -- it ran once and forgot. | Continuous reconciliation. ArgoCD compares cluster to Git every poll cycle and either reverts the drift (selfHeal: true) or surfaces it as OutOfSync. Drift is a first-class concept, not a postmortem discovery. |
| **Multi-cluster fan-out** | Linear: one job applies to one cluster, or you write a matrix. Each cell holds its own creds; failures partially-deploy and leave the fleet inconsistent. | Declarative: one Application (or ApplicationSet) per cluster, all reading the same Git state. The fleet *converges* on Git -- if 9 of 10 clusters succeeded, the 10th retries until it matches; no partial-state cleanup needed. |
| **Rollback semantics** | "Re-run CI with the previous SHA" -- requires CI history to exist, the previous artifact to still be in the registry, and someone to remember the SHA. Rollback time = full pipeline rerun (npm install + docker build + push + apply) ~10+ min, brittle on transient failures (registry rate-limits, unpublished deps, base-image mirror outage). | `git revert <commit>` in the config repo -- ArgoCD picks it up on the next poll and reconciles backward. Rollback is a Git operation, not a CI operation. Idempotent, deterministic, no human in the registry. **Rollback speed *is* a security property** -- every minute prod is broken costs revenue and trust, and a 10-minute CI-rebuild rollback is a different SLO than a 30-second `git revert`. |

Memorize the columns as the **five-axis defense** of GitOps CD. Every time an engineer says "but it would be simpler to just run `kubectl apply` from the workflow," walk them down the table. The "simpler" path costs you all five axes simultaneously, and you only notice on the day you need them.

### What "CI commits to Git" means concretely

The CI handoff is a *file mutation* on the config repo. There are precisely three legitimate forms it can take:

1. **CI direct-pushes to a tracked branch** -- typically only for dev: `yq -i '.image.tag = "..."' envs/dev/values.yaml && git commit -am "..." && git push`. ArgoCD's dev Application has `automated.prune: true` and reconciles within a poll cycle. This is fast and avoids PR ceremony for non-prod deploys.
2. **CI opens a PR via `peter-evans/create-pull-request`** -- typically for stg/prod: same `yq` edit, but the action wraps it in a branch + PR + body. A human reviews and merges; merge triggers ArgoCD reconciliation.
3. **CI does nothing; ArgoCD Image Updater watches the registry** -- write-back via the `git` method commits to the config repo on the cluster's behalf. CI's only job is to push the image; the manifest update happens controller-side.

Anything *else* CI does on the config repo is wrong. CI does not template Helm; CI does not apply manifests; CI does not run `kustomize build`. The repo's `values.yaml` files are the *only* artifact CI mutates. ArgoCD owns rendering and applying.

---

## Part 2: Image-Tag Update Patterns -- Three Approaches with a Decision Matrix

The three legitimate ways to bump the image tag in `dev/values.yaml`. Pick one per service. Mixing two is how the "two clerks scribbling over each other" gotcha (#3 in the gotcha list) actually happens in production.

### Approach 1: CI commits to the config repo

The most common pattern, and the right choice for **most teams under ~50 services**.

```yaml
# .github/workflows/release.yml -- final job after build/sign/push (May 7 doc)
jobs:
  build-and-sign:
    # ... May 7 doc: outputs.digest = sha256:abc...
    outputs:
      digest: ${{ steps.build.outputs.digest }}

  promote-to-dev:
    needs: build-and-sign
    runs-on: ubuntu-latest
    permissions:
      contents: read              # only reads app repo
      id-token: write              # OIDC for AWS (already used for ECR push)
    steps:
      - name: Generate GitHub App token for config repo
        id: app-token
        uses: actions/create-github-app-token@v2
        with:
          app-id: ${{ vars.CONFIG_REPO_APP_ID }}
          private-key: ${{ secrets.CONFIG_REPO_APP_PRIVATE_KEY }}
          owner: ${{ github.repository_owner }}
          repositories: my-config-repo

      - name: Checkout config repo
        uses: actions/checkout@v5
        with:
          repository: ${{ github.repository_owner }}/my-config-repo
          token: ${{ steps.app-token.outputs.token }}
          path: config
          fetch-depth: 0

      - name: Bump dev image digest
        working-directory: config
        env:
          NEW_DIGEST: ${{ needs.build-and-sign.outputs.digest }}
          IMG_REPO: ${{ vars.ECR_REPO }}
        run: |
          yq -i '.image.repository = strenv(IMG_REPO)' apps/myapp/envs/dev/values.yaml
          yq -i '.image.digest = strenv(NEW_DIGEST)' apps/myapp/envs/dev/values.yaml
          # Note: only update digest, NOT tag -- if the chart references both,
          # the digest wins at runtime but a stale tag in values is a footgun
          # (see gotcha #5).

      - name: Open PR (or push directly for dev)
        if: github.ref == 'refs/heads/main'
        uses: peter-evans/create-pull-request@v7
        with:
          path: config
          token: ${{ steps.app-token.outputs.token }}
          branch: bump/myapp-dev-${{ github.sha }}
          delete-branch: true
          commit-message: "chore(myapp): bump dev to ${{ needs.build-and-sign.outputs.digest }}"
          title: "Bump myapp dev image to ${{ needs.build-and-sign.outputs.digest }}"
          body: |
            Auto-bump from app repo SHA `${{ github.sha }}` (workflow run: ${{ github.run_id }}).

            - Image: `${{ vars.ECR_REPO }}@${{ needs.build-and-sign.outputs.digest }}`
            - Cosign-verifiable identity: `https://github.com/${{ github.repository }}/.github/workflows/release.yml@refs/heads/main`
          signoff: true
          labels: |
            automation
            env:dev
```

**Why a GitHub App token, not GITHUB_TOKEN.** There are *two* problems with `GITHUB_TOKEN` in this scenario, and both have to be solved:

1. **Scope.** Default `GITHUB_TOKEN` is auto-minted per workflow run and scoped to the *current* repo only. It cannot write to a different repo at all -- the config repo's API will return 403 / 404. So even before recursion enters the picture, `GITHUB_TOKEN` is structurally incapable of opening a cross-repo PR.
2. **Recursion fuse.** Even when you use a token that *can* cross-write (a PAT or App token) and you push or open a PR via Actions, **events authored by `GITHUB_TOKEN` itself do not cascade to other workflows in the same repo** by default -- a deliberate GitHub anti-recursion fuse to prevent runaway loops. PRs/pushes authored by a *PAT or App token* (not `GITHUB_TOKEN`) *do* trigger downstream workflows.

So the layering is: `GITHUB_TOKEN` fails on layer 1 (can't reach the other repo). A PAT clears layer 1 but couples the bot identity to a person (audit, rotation, off-boarding pain). A **GitHub App token** clears both layers cleanly: it has explicit cross-repo permissions, it's not bound to a human, and downstream workflows (the config repo's `lint`, `validate`, ArgoCD-pre-sync checks) actually fire so a green merge can't silently bypass your CI gate. Mint via `actions/create-github-app-token`. App tokens are the production-correct answer; PATs are an acceptable migration step.

**Why `digest` not `tag`.** May 7 hammered "sign by digest, never by tag." The same logic applies to GitOps: the values.yaml entry should pin the digest (`image.digest: sha256:abc...`), so the cluster pulls the bytes you signed, not whatever bytes a (possibly compromised) tag points at later. If your Helm chart references both `image.tag` and `image.digest`, configure the chart so the digest wins (most charts use `{{ .Values.image.repository }}@{{ .Values.image.digest }}` when digest is set). Updating both leads to "tag says v1.2.3, digest says abc, neither matches" debugging pain (gotcha #5).

**The audit story this gives you.** The PR has the SHA of the app-repo commit in the body, the digest in the title, and the workflow run linked. `git log -- envs/prod/values.yaml` in the config repo gives you "every change that ever hit prod, in chronological order, with the merging human's name and the upstream commit." That's the full audit trail in two `git log` invocations -- no CI-history export, no registry tag archaeology.

### Approach 2: ArgoCD Image Updater

Image Updater is a separate controller (deployed alongside ArgoCD) that polls the registry and writes the new tag/digest **into Git** (or, in default config, into the live Application). The robot manifest-clerk. Right choice when **you don't want CI to know about the config repo at all** -- e.g., the app team owns the image build, the platform team owns the config repo, and you'd rather not give CI cross-repo App tokens.

The Application annotations:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-dev
  namespace: argocd
  annotations:
    # 1. Watch list -- one entry per image. The alias `myapp` is referenced below.
    argocd-image-updater.argoproj.io/image-list: myapp=123.dkr.ecr.us-east-1.amazonaws.com/myapp

    # 2. Strategy -- track a mutable tag (main) by digest. Pairs with main-branch CI
    #    that re-pushes :main after every merge.
    argocd-image-updater.argoproj.io/myapp.update-strategy: digest
    argocd-image-updater.argoproj.io/myapp.allow-tags: regexp:^main$

    # 3. Helm value paths -- where in values.yaml to write. Note these are paths,
    #    NOT keys; the most common copy-paste error is using `repository:` instead of
    #    `image.repository`. Mismatch is silent (gotcha #13).
    #
    #    DIGEST-MODE NOTE: with `update-strategy: digest`, the *value* written here is
    #    a sha256 digest, not a semver tag. Stuffing a digest into a key called
    #    `helm.image-tag` works but conflates slot semantics. The cleaner, version-
    #    safe options are:
    #      (a) `<alias>.helm.image-spec: image.fqin` -- write the combined
    #          "repo@sha256:abc..." (fully-qualified image name) into a single key.
    #          Your chart then templates `image: {{ .Values.image.fqin }}`.
    #      (b) `manifestTargets.helm.digest` mapping (Image Updater v0.13+) -- maps
    #          repository, tag, and digest to three separate values.yaml keys.
    #    The form below works on all versions but the digest lives under a
    #    `helm.image-tag`-named annotation -- read it as "the field Image Updater
    #    will write the new identifier to," not "this is necessarily a tag."
    argocd-image-updater.argoproj.io/myapp.helm.image-name: image.repository
    argocd-image-updater.argoproj.io/myapp.helm.image-tag: image.digest
    # `allow-tags` is effectively MANDATORY in digest mode -- without it, Image Updater
    # has no way to pick which mutable tag's digest to track and behavior is non-deterministic.
    # Below: track only the `main` tag's digest; new push to `main` -> new digest -> new commit.

    # 4. Write-back -- always `git`, never `argocd` (see drift trap below).
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/write-back-target: helmvalues:./apps/myapp/envs/dev/values.yaml

    # 5. Branch -- MUST match Application's targetRevision or write-back goes
    #    to the wrong branch silently (gotcha #4).
    argocd-image-updater.argoproj.io/git-branch: main

    # 6. ECR-specific: pull credentials via auth script (IAM role assumption can't
    #    be expressed as a static secret; the configmap maps `pullsecret:` to a
    #    shell script that calls `aws ecr get-login-password`).
    argocd-image-updater.argoproj.io/myapp.pull-secret: ext:/scripts/auth1.sh
spec:
  project: myapp
  source:
    repoURL: https://github.com/my-org/my-config-repo.git
    targetRevision: main             # MUST equal git-branch annotation above
    path: apps/myapp/envs/dev
    helm:
      valueFiles: [values.yaml]
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**The `argocd` vs `git` write-back trap.** Image Updater's *default* write-back method is `argocd`, which writes parameter overrides via the ArgoCD API straight into the live Application's `spec.source.helm.parameters`. **This drifts from Git the moment a human edits values.yaml.** The Application's effective state is "values.yaml + live override," and now `git diff` lies about what's running. Worse, deleting and recreating the Application wipes the override, so your image tag history vanishes the moment someone runs `kubectl delete app`. **Always set `write-back-method: git`** -- it commits a real file change that survives App lifecycle and shows up in `git log`. This is the single highest-impact configuration line in any Image Updater setup.

**Branch tracking.** The `git-branch` annotation must equal the Application's `targetRevision`. If the Application tracks `main` but `git-branch` is `image-updates`, the bot commits to `image-updates` -- which ArgoCD never reads -- and your tag bumps go into a parallel branch nobody syncs. Silent failure, no error log, just "the cluster doesn't update." (Gotcha #4.)

**Helm path mismatch.** `helm.image-name: image.repository` writes to `image.repository` in your `values.yaml`. If your chart actually consumes `app.image.repo` (different key), Image Updater happily writes to the wrong key, and the chart's template never sees the new digest. The bug is silent: no error, no log, just the cluster keeps running the old image. Test with `argocd-image-updater test` (CLI, runs the resolver against your annotation set without pushing) before deploying. (Gotcha #13.)

#### The diagnostic command you'll reach for every time

The single most useful Image Updater debug tool is the `test` subcommand. It runs the resolver against any image+strategy combination and tells you what *would* be written, without actually committing or touching the live Application. Memorize this invocation:

```bash
# Inside the controller pod (so it has registry creds)
kubectl exec -n argocd deploy/argocd-image-updater -- \
  argocd-image-updater test \
    123.dkr.ecr.us-east-1.amazonaws.com/payments \
    --update-strategy digest \
    --allow-tags 'regexp:^main$'

# Output you want: "rolling out new digest sha256:abc... for image payments"
# Output that means broken: "no candidate found" / "tag latest not found in registry"
```

Run this first whenever someone says "Image Updater isn't updating." Half the time the answer is `image-list` is missing a tag specifier (digest mode falls back to implicit `:latest`, which CI never pushes), `allow-tags` is missing in digest mode (non-deterministic candidate selection), or `helm.image-name` points at a key the chart doesn't consume. All three are silent no-ops; `test` makes them loud.

#### Incident runbook: pause Image Updater BEFORE you edit Git

A common operational mistake during a bad-image rollback: the on-call edits `values.yaml` to revert to the previous digest, commits, pushes -- and Image Updater overwrites the revert on its next poll cycle (default 2 min) because the registry's mutable tag (e.g., `:main`) still points at the bad build. The revert wins for 90 seconds, then the controller "fixes" the file back to the bad digest. The on-call thinks they're losing their mind.

**Correct order:**

1. **Pause Image Updater for the affected image first.** Annotate the Application with `argocd-image-updater.argoproj.io/<alias>.ignore-tags: '*'` (matches everything, controller skips), or remove the `image-list` annotation entirely. The controller stops polling for that image immediately.
2. *Then* `git revert` (or manual edit) the values file. ArgoCD reconciles backward with no fight.
3. After the bad build is fixed and a new known-good image exists in the registry, **re-enable Image Updater** by restoring the original annotations.

This is the operational complement to the `argocd` vs `git` write-back fix: even with `git` write-back, Image Updater is still a daemon that *will* re-push the registry's current head to your values file. Pause first, fix second, re-enable third.

### Approach 3: Promotion engine -- Kargo or GitOps Promoter

Kargo (Akuity, CNCF Sandbox) and GitOps Promoter (argoproj-labs) are both **dedicated promotion controllers**. They model environments as a graph of `Stage` CRDs, ingest immutable artifact bundles (`Freight` in Kargo) from registries/Git/charts, and orchestrate per-stage promotion with verification gates. The freight broker.

```yaml
# Kargo Warehouse -- subscribes to image registry
apiVersion: kargo.akuity.io/v1alpha1
kind: Warehouse
metadata:
  name: myapp
  namespace: myapp
spec:
  subscriptions:
    - image:
        repoURL: 123.dkr.ecr.us-east-1.amazonaws.com/myapp
        semverConstraint: ^1.0.0
---
# Kargo Stage -- one per environment
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: dev
  namespace: myapp
spec:
  requestedFreight:
    - origin:
        kind: Warehouse
        name: myapp
      sources:
        direct: true               # auto-promote latest freight to dev
  promotionTemplate:
    spec:
      steps:
        - uses: git-clone
          config:
            repoURL: https://github.com/my-org/my-config-repo.git
            checkout:
              - branch: main
                path: ./out
        - uses: helm-update-image
          config:
            path: ./out/apps/myapp/envs/dev/values.yaml
            images:
              - image: 123.dkr.ecr.us-east-1.amazonaws.com/myapp
                key: image.digest
                value: ImageAndDigest
        - uses: git-commit
          config:
            path: ./out
            messageFromSteps: [helm-update-image]
        - uses: git-push
          config:
            path: ./out
        - uses: argocd-update
          config:
            apps:
              - name: myapp-dev
  verification:
    analysisTemplates:
      - name: kargo-canary-check
---
# Kargo Stage -- staging, requires dev verification to pass first
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: stg
  namespace: myapp
spec:
  requestedFreight:
    - origin: { kind: Warehouse, name: myapp }
      sources:
        stages: [dev]              # only freight that passed dev's verification
  promotionTemplate:
    # ... same steps, different path
  verification:
    analysisTemplates:
      - name: kargo-slo-check
```

When to reach for Kargo: **~5 environments × ~10 services × multi-region = ~150 deploy combinations** is the line where `promote-stg.yml` × `promote-prod.yml` × N service repos becomes unmaintainable. Kargo collapses that into one CRD per stage, one Warehouse per artifact source, and a single UI showing "freight `sleepy-swan-123` is in dev + stg, blocked at prod by failed verification." Read the May 7 syllabus → freecodecamp tutorial for a full hands-on Kargo setup; today's purpose is to know *where it sits* in the decision matrix below.

### Decision matrix: which image-update mechanism

| Factor | CI commits PR | Image Updater (`git`) | Kargo |
|---|---|---|---|
| **Team size** | 1-50 services | 1-200 services | 50+ services |
| **Deploy frequency** | <50/day | <100/day | Any |
| **Multi-env fan-out** | Each env = a separate workflow | Each env = a separate Application | Each env = a Stage; native graph |
| **Audit-trail strictness** | Highest -- Git PR + CI run + reviewer name | Medium -- Git commit by `argocd-image-updater` bot | High -- Promotion CRD + verification record |
| **Cross-repo coupling** | CI knows about config repo | CI knows nothing about config repo | CI knows nothing about config repo |
| **Hand-rollable in a weekend?** | Yes | Yes (annotations only) | No -- install + Stage CRDs + Argo Workflows for verification |
| **Verification gates** | GitHub Environments + manual review | None native | First-class (`AnalysisTemplate` per Stage) |
| **Best for** | Team default, app teams own deploy | Platform-team-owned config repo, separation of concerns | Multi-region, multi-service, SLO-gated promotion |

**The wrong answer is mixing two.** Image Updater + CI both pushing to the same `dev/values.yaml` is a guaranteed merge conflict storm, with the bonus that "who pushed this commit" debugging now spans two automations. Pick one mental model per service; document it in the service's `README.md`.

---

## Part 3: Repo Topology -- App Repo vs Config Repo vs Single Repo

The decision that locks in your permission model, your audit granularity, and your blast radius for the next 18 months. Three shapes, and only one of them is the modern default.

### The three shapes

```
TWO-REPO (app + config) -- THE PRODUCTION DEFAULT
  app-repo:    src/, Dockerfile, charts/, .github/workflows/release.yml
  config-repo: apps/myapp/envs/{dev,stg,prod}/values.yaml, appsets/, .github/workflows/validate.yml
  ArgoCD points at config-repo only.
  CI in app-repo cross-writes to config-repo via GitHub App token.

SINGLE-REPO (mono) -- SMALL TEAMS, FAST ITERATION
  one-repo:    src/, Dockerfile, charts/, envs/{dev,stg,prod}/values.yaml, .github/workflows/
  ArgoCD points at the same repo it builds from (different paths).
  CI commits image-tag updates to envs/dev/values.yaml in the same repo.

THREE-REPO (per-environment-config) -- ANTI-PATTERN, AVOID
  app-repo, config-dev, config-stg, config-prod -- separate repos per env
  Promotion = cross-repo-cherry-pick. Maximum drift. No one does this in 2026.
```

### Two-repo: why it's the default

1. **Permission boundaries.** Engineering team has write on `app-repo`; platform/SRE team has write on `config-repo`. The line between "code change" and "deploy approval" is enforced by GitHub Teams, not by a CODEOWNERS regex inside one repo. A junior engineer can ship code to staging without ever having permission to mutate prod's `values.yaml`.
2. **Audit-trail granularity.** `git log` of `config-repo` is a clean record of "every deploy that ever happened across all envs." You don't have to filter app-code commits out of it. For SOC 2 / ISO 27001 audits, this is the artifact the auditor wants -- one repo, deploy-only history, every commit signed.
3. **Rollback granularity.** `git revert` on the config repo rolls back exactly one deploy without touching app code. In a single repo, "revert this deploy" requires path-scoped reverts (`git revert -- envs/prod/`), which is brittle when the same commit touched both code and config.
4. **CI scope minimization.** App-repo CI only needs registry-write + cross-repo App token. Config-repo CI only needs validation tools (helm template, kubectl --dry-run, conftest). Neither holds cluster credentials. Compromising either repo's CI does not give you compromised cluster access.
5. **Multi-tenant ArgoCD.** ArgoCD's per-repo credentials are simpler when each tenant has one config repo, even if they share many app repos.

### Single-repo: when it's right

For **small teams (<10 engineers, <5 services)** with a single deploy target, the two-repo overhead is real: every PR that changes both code and deploy config now needs two PRs, one in each repo. A single repo with `envs/` subdirectories collapses that. Trade-offs you're accepting: weaker permission boundaries, mixed audit timelines, and no separation between "code reviewer" and "deploy approver." Most startups start single-repo and split when they hit ~5-10 engineers; that split is a recognized growth event, not a regression.

### Three-repo (per-env): why it's an anti-pattern

The instinct is "prod's config is sensitive, so let it live in its own repo with its own protection rules." The execution is misery: promoting a tag from dev means cherry-picking a commit between repos, your config drifts because each repo evolves independently, and `git log` no longer tells you "what state was prod in last Tuesday" because that's three repo-cross-references. The same security goal is achieved by **one config repo with `CODEOWNERS` per env folder** + GitHub Environments protection rules + branch protection rules. Modern verdict: don't.

### Folder-per-environment vs branch-per-environment vs repo-per-environment

The April 19 doc covered this; today it pairs with the recursion gotcha below.

| Layout | What promotion looks like | Anti-pattern? |
|---|---|---|
| **Folder-per-env** (`envs/dev/`, `envs/stg/`, `envs/prod/` on `main`) | `cp envs/dev/values.yaml envs/stg/values.yaml`, then PR. One commit, perfectly diffable. | NO -- modern default (Kapelonis 2022, still load-bearing in 2026) |
| **Branch-per-env** (`dev`, `stg`, `prod` long-lived branches) | `git checkout stg && git merge dev` -- drags every dev iteration into stg, drift over time, hot-fix cherry-picks fragment history. | YES -- the canonical wrong answer |
| **Repo-per-env** (separate repos) | Cross-repo cherry-pick. Drift across all 3 repos with no diff visibility. | YES -- maximum overhead, minimum benefit |

### Four config categories -- the separation rule (Kapelonis)

In your folder-per-env layout, **never put all four categories in one file.** They get promoted on different cadences:

| Category | Examples | Promotion cadence |
|---|---|---|
| **App-config (image versions)** | `image.digest`, `image.tag`, chart `appVersion` | Per release -- the thing CI auto-bumps |
| **Infra-config (platform settings)** | replica count, resource limits, HPA thresholds, service-mesh annotations | Quarterly -- per cluster sizing review |
| **Env-overlays (static business config)** | API endpoints per region, feature-flag-set defaults, S3 bucket names | Monthly -- per env-introduction |
| **Release-promotion files (just-image)** | `version.yaml` containing only `image.digest: ...` | Per release -- the file Kapelonis recommends `cp`-ing for promotion |

Mix all four into `envs/prod/values.yaml` and "promote dev to stg" becomes "copy a 200-line file that includes replica counts and feature flags," which silently overrides stg's sizing every release. **The Kapelonis pattern: one `values.yaml` per env for static config, plus a tiny `version.yaml` per env that *only* holds the image digest -- promote by copying just `version.yaml`.**

### The recursion / infinite-loop trap

CI commits to config repo -> config repo's CI fires -> if the config repo's CI also rebuilds something that triggers the app repo's CI -> infinite loop.

**Important:** unlike GitLab CI / CircleCI / Jenkins, **GitHub Actions does NOT natively honor `[skip ci]` in commit messages**. The marker is just a string -- GHA will run the workflow anyway. You have to *explicitly* short-circuit:

- **Path filters** (preferred): scope the config repo's CI to `paths: ['.github/**', 'scripts/**']` and exclude `apps/**/values.yaml` -- bot commits to values files don't trigger CI at all.
- **Commit-message guard at the job level**: `if: "!contains(github.event.head_commit.message, '[skip ci]')"` on every job. Verbose; easy to forget on new jobs.
- **Bot-author guard**: `if: github.actor != 'argocd-image-updater[bot]' && github.actor != 'deploy-bot[bot]'`. Robust; one place; doesn't depend on commit-message convention.

In practice: use `paths-ignore` as the primary fence and bot-author guards as defense in depth. Keep `[skip ci]` in your bot commit messages anyway -- it's the cross-tool convention, it's harmless on GHA, and the same config repo may also be processed by tools that *do* honor it (Renovate, Dependabot, GitLab mirrors).

The subtler trap: **`peter-evans/create-pull-request` requires a PAT or GitHub App token, and there are two reasons stacked on top of each other.** First, `GITHUB_TOKEN` is repo-scoped and structurally cannot write to a different repo -- if you're following the two-repo pattern (app repo opens PRs against config repo), the call fails with 403/404. Second, even for *same-repo* writes, **events authored by `GITHUB_TOKEN` do not trigger downstream workflows** -- GitHub's anti-recursion fuse. So a PR opened by `GITHUB_TOKEN` looks like a "draft PR that bypasses CI": no `lint`, no `validate`, no required-status-check evaluation. A GitHub App token (or PAT) clears both layers. Mint via `actions/create-github-app-token`. (Fine-grained PATs work but tie a bot's identity to a person.) Don't disable the recursion fuse -- route around it with explicit cross-repo App auth.

---

## Part 4: GitHub Environments -- The Customs Checkpoint

GitHub Environments is the single most-underused primitive in modern GitHub Actions setups. Most teams stop at "set required reviewers, click save" and miss the four other levers that make Environments the *security boundary* between CI and prod, not just an approval popup.

### The four protection rules

```yaml
# Job that targets an environment
deploy-prod:
  needs: build
  runs-on: ubuntu-latest
  environment:
    name: production            # enables protection rules; exposes env-scoped secrets
    url: https://argocd.example.com/applications/myapp-prod   # populates Deployments sidebar
  permissions:
    id-token: write
    contents: read
  steps:
    - name: Configure AWS (env-scoped role)
      uses: aws-actions/configure-aws-credentials@v6
      with:
        role-to-assume: ${{ vars.PROD_DEPLOY_ROLE_ARN }}
        aws-region: us-east-1
    - run: ./scripts/promote.sh prod
```

**Rule 1 -- Required reviewers.** Up to **6** users or teams; only one needs to approve; reviewers must have at least read access; the job sits in `waiting` state and the listed reviewers receive a UI banner + email.

**Critical security boundary: environment-scoped secrets are NOT injected into the job until approval is granted.** A malicious PR cannot exfiltrate prod secrets even if it triggers the deploy workflow, because the secrets aren't materialized in the runner's env until a human approves. This is what makes "fork PRs that target environment: production" safe-by-design -- the worst a malicious fork can do is wait in the approval queue.

**Self-review.** Until late 2024, a reviewer could approve a deploy they themselves triggered. The new "Prevent self-review" toggle (per environment) blocks this. **Enable it on production environments.** (Gotcha #6.)

**Rule 2 -- Wait timer.** 1 to **43,200 minutes** (30 days max). Composes additively with required reviewers (you wait the timer *and* get reviewer approval). Use case: "let the canary soak for 30 minutes after stg before letting prod start." **Wait time is not billable** -- a 24-hour wait timer doesn't burn 24 hours of Actions minutes (gotcha #7 caveat: the *gating job* may still cost a small amount in some patterns, but the wait itself is free).

**Rule 3 -- Deployment branch and tag policies.** Restrict which refs can deploy to this environment. Two policy types, available in the same UI but **they don't compose with AND semantics across types** -- they list refs that *match* the policy. The branch policy options:
- **All branches** -- default, no restriction.
- **Protected branches only** -- any branch covered by a branch-protection rule.
- **Selected branches/tags** -- explicit allowlist with name patterns (`main`, `release/*`).

A 2024 update added **tag policies** as a separate concern. You can configure both branches and tags; a deployment qualifies if it matches *any* policy of the appropriate type. (Gotcha #8: "I configured branch policy *and* tag policy; my main-branch deploy was rejected." Resolution: ensure the matching kind has at least one rule that matches; both kinds together act as an OR within their type, but a deploy must match the type that actually triggered it.)

**Rule 4 -- Custom deployment protection rules** (GA 2024). Third-party gating via GitHub Apps. Examples: ServiceNow change-ticket validation, Datadog SLO health check, your own webhook receiver. The deployment is paused, GitHub fires a webhook to the configured App with a callback URL, the App calls back with `approve` or `reject`. Use case: "no production deploy starts unless the company's change-management ticket is in `approved` state." This is the integration point you wire when InfoSec mandates change-control compliance.

### Plan caveat

Required reviewers + wait timers + deployment branch policies are **GitHub Free for public repositories**, **GitHub Pro / Team / Enterprise for private repositories**. (Verified 2026.) Most production teams hit this surprise the first time they try to gate a private monorepo on a Free plan -- check the plan before architecting around environments. **Custom deployment protection rules** are stricter still: available on **public repos on all plans**, but on **private repos they require GitHub Enterprise** (not Pro / Team). If you've designed a workflow around ServiceNow / Datadog gating but you're on Team for a private repo, the integration just won't work -- budget Enterprise or fall back to a manual webhook receiver.

### The `environment:` key on a job -- what it actually does

```yaml
deploy-prod:
  environment: production      # implicit form, no URL
  # OR
  environment:
    name: production
    url: ${{ steps.deploy.outputs.argocd-url }}
```

When a job declares `environment:`, GitHub does five things:

1. **Creates a deployment** via the Deployments API for that environment. This populates the repo's "Environments" sidebar.
2. **Gates the job on protection rules.** If reviewers/wait/branch policy aren't satisfied, the job is `waiting`, then `queued`, then `in_progress`.
3. **Exposes environment-scoped secrets and variables** to the job. Repo-scoped + env-scoped secrets compose: env wins on name collision.
4. **Sets the OIDC `sub` claim** to `repo:org/repo:environment:<name>` for that job's OIDC token. (See next subsection.)
5. **Fires deployment status events** (`queued`, `in_progress`, `success`, `failure`) -- consumable by GitHub Apps and the Deployments API.

### Environment-scoped secrets vs repo-scoped vs org-scoped -- precedence

```
Job's secret resolution order (highest precedence first):
  1. Environment-scoped secrets (only when job declares `environment:`)
  2. Repository-scoped secrets
  3. Organization-scoped secrets (filtered by repo-access policy)
```

Two foot-guns:

- **Environment-scoped secret on a job that doesn't declare `environment:`** -- secret is **not exposed**, no error, just silently empty (`secrets.PROD_DEPLOY_KEY` evaluates to ``). The job runs, fails on the missing creds, and the engineer spends an hour wondering why the secret "isn't loading." (Gotcha #9.)
- **Same secret name at repo and env scope.** Env wins. Useful for "different DATABASE_URL per env from the same secret name," but be deliberate about it -- accidentally creating an env-scoped `AWS_REGION` that overrides a repo-scoped one is a confusing rename hunt.

### The OIDC `sub` claim in the environment context

This is the line that connects May 7's OIDC content to today's GitHub Environments content. The `sub` format depends on what triggered the job:

| Trigger | `sub` claim |
|---|---|
| Push to a branch (no `environment:`) | `repo:org/repo:ref:refs/heads/<branch>` |
| Push to a tag (no `environment:`) | `repo:org/repo:ref:refs/tags/<tag>` |
| Pull request (no `environment:`) | `repo:org/repo:pull_request` (DANGEROUS, see May 7) |
| Job declaring `environment: production` (any trigger) | `repo:org/repo:environment:production` |

**The environment-scoped `sub` is the production-correct trust-policy condition for any role that touches prod.** Trust policy:

```json
{
  "Effect": "Allow",
  "Principal": { "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com" },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
      "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:environment:production"
    }
  }
}
```

What this enforces:

1. The workflow MUST declare `environment: production` to mint a credential.
2. GitHub MUST have already evaluated the environment's protection rules: required reviewers, wait timer, branch policy, custom rules.
3. The trust-policy boundary is "no prod credential is ever issued without GitHub having already enforced human approval."

The two-line difference -- `ref:refs/heads/main` vs `environment:production` -- is the difference between "any push to main can take down prod" and "prod requires a customs stamp." It's the most cost-effective security upgrade in the CI/CD playbook.

### The promotion-trust-policy mismatch (gotcha #10)

When a workflow runs without `environment:` on a push to main, the `sub` is `...:ref:refs/heads/main`. When that *same* workflow file later runs *with* `environment: production`, the `sub` becomes `...:environment:production`. **A trust policy scoped to `ref:refs/heads/main` will reject the second run** with `Not authorized to perform sts:AssumeRoleWithWebIdentity`. The fix is to use *separate roles* per phase: a `BuildRole` whose trust accepts `ref:refs/heads/main` for the CI build/push, and a `DeployProdRole` whose trust accepts `environment:production` for the deploy job. Each role has minimum-necessary permissions; OIDC scoping mirrors the handoff.

### Deployment status events -- what fires when

When a job declares `environment:`, GitHub fires deployment status webhooks:

| State | When it fires | Consumed by |
|---|---|---|
| `queued` | Job dispatched, awaiting protection rules / runner | (rare, for rate-limit displays) |
| `pending` | Job is queued for execution | Custom protection rule callbacks |
| `in_progress` | Runner picked up the job, protection rules satisfied | Deployments API listeners (Slack bots, dashboards) |
| `success` | Job completed successfully | ArgoCD Notifications can ALSO fire this on Application healthy |
| `failure` | Job failed | Deployments sidebar shows red marker |

The pattern in the May 7 doc + today: app-repo CI fires `success` on the deployment when the config-repo PR merges; ArgoCD Notifications then fires *another* `success` on the same deployment when the cluster reconciles. The Deployments sidebar shows "two green dots" -- one for "manifest committed" and one for "cluster updated." Most teams alias the second one as the canonical "deployed" event.

---

## Part 5: Promotion Workflow -- dev → stg → prod

Three concrete patterns, each with full YAML. **Most teams use a combination** -- auto-promote dev, PR-promote stg, tag-driven prod -- not a single pattern across all envs.

### Pattern A: Auto-promote dev, PR-promote stg/prod

The bread-and-butter pattern. Every push to `main` in the app repo auto-deploys to dev; promoting to stg/prod requires a PR review.

```yaml
# In the app repo: .github/workflows/release.yml (excerpt)
name: Release
on:
  push:
    branches: [main]

jobs:
  build-and-push:
    # ... May 7 build/sign/push, outputs.digest
    outputs:
      digest: ${{ steps.build.outputs.digest }}

  promote-dev:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment:                    # GitHub Environment "dev" -- no protection rules; populates Deployments
      name: dev
      url: https://argocd.example.com/applications/myapp-dev
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/create-github-app-token@v2
        id: token
        with:
          app-id: ${{ vars.CONFIG_REPO_APP_ID }}
          private-key: ${{ secrets.CONFIG_REPO_APP_PRIVATE_KEY }}
          owner: ${{ github.repository_owner }}
          repositories: my-config-repo
      - uses: actions/checkout@v5
        with:
          repository: ${{ github.repository_owner }}/my-config-repo
          token: ${{ steps.token.outputs.token }}
      - name: Direct-push to dev
        env:
          NEW_DIGEST: ${{ needs.build-and-push.outputs.digest }}
        run: |
          yq -i '.image.digest = strenv(NEW_DIGEST)' apps/myapp/envs/dev/values.yaml
          git config user.email "deploy-bot@my-org.com"
          git config user.name "Deploy Bot"
          git add apps/myapp/envs/dev/values.yaml
          git commit -m "chore(myapp): bump dev to ${NEW_DIGEST} [skip ci]"
          # NOTE: [skip ci] is a cross-tool convention (GitLab/Circle honor it) but
          # GitHub Actions does NOT respect it natively. The actual recursion fence
          # is the config repo's `paths-ignore: ['apps/**/values.yaml']` workflow
          # filter plus a bot-author `if: github.actor != 'deploy-bot[bot]'` guard.
          git push origin main
```

```yaml
# In the config repo: .github/workflows/promote-stg.yml
name: Promote dev to stg
on:
  workflow_dispatch:
  schedule:
    - cron: '0 14 * * 1'           # Mondays 14:00 UTC: weekly stg promotion offer

jobs:
  open-promotion-pr:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v5
      - name: Compute current dev digest
        id: dev
        run: echo "digest=$(yq '.image.digest' apps/myapp/envs/dev/values.yaml)" >> $GITHUB_OUTPUT
      - name: Update stg values to match dev
        run: |
          yq -i ".image.digest = \"${{ steps.dev.outputs.digest }}\"" apps/myapp/envs/stg/values.yaml
      - uses: peter-evans/create-pull-request@v7
        with:
          branch: promote/stg-myapp
          delete-branch: true
          commit-message: "chore(myapp): promote dev to stg (${{ steps.dev.outputs.digest }})"
          title: "Promote myapp dev → stg"
          body: |
            Promoting dev image to stg.
            - Digest: `${{ steps.dev.outputs.digest }}`
            - Diff: see file changes below
          labels: env:stg, promotion
          reviewers: my-org/sre, my-org/eng-lead
```

The stg PR's merge then re-triggers the same workflow file on `main` (via a `paths:` filter), which checks the stg files for changes and opens the prod-promotion PR. Each environment is one PR up the chain; each merge is one human approval.

### Pattern B: Tag-driven prod

For teams that want **prod deploys to require an explicit `git tag v1.2.3`** rather than a PR merge. The tag is the trigger.

```yaml
name: Promote to prod (tag-driven)
on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'    # only semver -- no v1.2.3-rc1 pre-release
      - '!v[0-9]+.[0-9]+.[0-9]+-*' # exclude pre-release suffixes

jobs:
  validate-tag:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
        with:
          fetch-depth: 0           # MANDATORY for tag history (gotcha #14)
      - name: Verify tag matches a stg-deployed digest
        run: |
          # Cross-check: does this tag's image already exist in stg?
          STG_DIGEST=$(yq '.image.digest' apps/myapp/envs/stg/values.yaml)
          TAG_DIGEST=$(crane digest 123.dkr.ecr.../myapp:${{ github.ref_name }})
          if [ "$STG_DIGEST" != "$TAG_DIGEST" ]; then
            echo "::error::Tag ${{ github.ref_name }} (digest $TAG_DIGEST) does not match stg digest ($STG_DIGEST). Promote to stg first."
            exit 1
          fi

  promote-prod:
    needs: validate-tag
    runs-on: ubuntu-latest
    environment:
      name: production              # gated: required reviewers, wait timer 30m, branch:tags-only
      url: https://argocd.example.com/applications/myapp-prod
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v5
      - run: |
          yq -i ".image.digest = \"$(crane digest 123.dkr.ecr.../myapp:${{ github.ref_name }})\"" \
            apps/myapp/envs/prod/values.yaml
      - uses: peter-evans/create-pull-request@v7
        with:
          branch: promote/prod-${{ github.ref_name }}
          commit-message: "chore(myapp): promote ${{ github.ref_name }} to prod"
          title: "Promote myapp ${{ github.ref_name }} → prod"
          body: |
            Tag-driven prod promotion.
            - Tag: `${{ github.ref_name }}`
            - Required: review by SRE on-call + 30-minute soak after stg
```

`fetch-depth: 0` is **non-optional** -- without it, `actions/checkout@v5` shallow-clones with depth 1, which means `crane digest` against a previous tag fails because the runner's local repo doesn't know about earlier commits. (Gotcha #14.)

### Pattern C: Workflow_dispatch with `environment` input

For ad-hoc / re-deploy scenarios. The operator picks the environment from a dropdown.

```yaml
name: Manual deploy
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options: [dev, stg, prod]
      digest:
        description: 'Image digest (sha256:...)'
        required: true
        type: string

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}    # dynamic; uses environment by name
    permissions:
      id-token: write
      contents: write
    steps:
      - uses: actions/checkout@v5
      - run: |
          yq -i ".image.digest = \"${{ inputs.digest }}\"" apps/myapp/envs/${{ inputs.environment }}/values.yaml
          git config user.email deploy-bot@my-org.com
          git config user.name "Deploy Bot"
          git commit -am "chore(myapp): manual deploy ${{ inputs.digest }} to ${{ inputs.environment }} [skip ci]"
          git push
```

The `environment: ${{ inputs.environment }}` line means *the same workflow* gates differently per env -- prod gets reviewer + wait, dev gets neither. The OIDC `sub` follows along (`environment:dev`, `environment:stg`, `environment:prod`), so you need one trust-policy per env (Pattern below in Part 10's Terraform).

### Where promotion gates live -- and why most teams need both

Three layers of gating exist in a mature pipeline:

| Gate | Layer | What it enforces | When to use |
|---|---|---|---|
| **GitHub Environments** | CI side | Reviewer approval BEFORE workflow runs the deploy step | The CI/CD handoff -- "no manifest is ever written to prod without a human" |
| **ArgoCD AppProject** | CD side | Restricts which sources/destinations/clusters this Application can target | Prevent ArgoCD itself from being weaponized to deploy unrelated workloads to prod |
| **Kargo Stage** | Promotion engine | Multi-leg promotion with verification (canary, SLO check) | When >5 envs and verification matters -- replaces ad-hoc workflow files |

**Most teams need GitHub Environments AND ArgoCD AppProject.** They protect different threats:
- Environments stops a malicious / careless PR from triggering a prod deploy.
- AppProject stops a compromised ArgoCD (or a misconfigured Application) from deploying *anything* to prod that wasn't pre-approved.

Belt-and-suspenders. Each layer's failure mode is different; either alone leaves a hole.

---

## Part 6: Closing the Loop -- ArgoCD → GitHub Notifications

The CI pipeline's green check tells you the *manifest was written*. The cluster's reconciliation tells you the *workload is running*. Without a feedback path from CD back to CI, the green check is a vanity metric -- "deployed" is not "running healthy."

ArgoCD Notifications closes the loop. It has been **built into ArgoCD core since v2.3** (do not install the standalone `argocd-notifications` repo -- it's archived).

### Configure the GitHub service

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.github: |
    appID: $github-appid                 # GitHub App App-ID
    installationID: $github-installid    # per-installation
    privateKey: $github-privatekey       # PEM, stored as a secret
  trigger.on-deployed: |
    - description: Application is synced and healthy
      send: [github-deployed]
      when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
  trigger.on-health-degraded: |
    - description: Application has degraded health
      send: [github-degraded]
      when: app.status.health.status == 'Degraded'
  trigger.on-sync-failed: |
    - description: Application sync failed
      send: [github-sync-failed]
      when: app.status.operationState.phase in ['Error', 'Failed']
  template.github-deployed: |
    message: Application {{.app.metadata.name}} synced and healthy.
    github:
      repoURLPath: '{{(call .repo.GetAppDetails).Helm.ValueFiles | toJson}}'
      revisionPath: '{{.app.status.sync.revision}}'
      status:
        state: success
        label: 'argocd/{{.app.metadata.name}}'
        targetURL: '{{.context.argocdUrl}}/applications/{{.app.metadata.name}}'
      deployment:
        state: success
        environment: '{{.app.metadata.labels.env}}'
        environmentURL: '{{.context.argocdUrl}}/applications/{{.app.metadata.name}}'
        logURL: '{{.context.argocdUrl}}/applications/{{.app.metadata.name}}?operation=true'
        requiredContexts: []
        autoMerge: true
        transientEnvironment: false
      pullRequestComment:
        content: |
          Application {{.app.metadata.name}} deployed:
          - Revision: `{{.app.status.sync.revision}}`
          - Health: ✅ {{.app.status.health.status}}
          - URL: {{.context.argocdUrl}}/applications/{{.app.metadata.name}}
  template.github-degraded: |
    message: Application {{.app.metadata.name}} health is degraded
    github:
      status:
        state: failure
        label: 'argocd/{{.app.metadata.name}}'
        targetURL: '{{.context.argocdUrl}}/applications/{{.app.metadata.name}}'
```

### Wire it on the Application

```yaml
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-deployed.github-deployed: ""
    notifications.argoproj.io/subscribe.on-health-degraded.github-degraded: ""
    notifications.argoproj.io/subscribe.on-sync-failed.github-sync-failed: ""
  labels:
    env: prod
```

### Three GitHub-native event types

The Notifications GitHub service can post to three GitHub APIs simultaneously:

1. **Commit statuses** -- `POST /repos/{owner}/{repo}/statuses/{sha}`. The green/yellow/red dot next to the commit. Appears in the PR's "Checks" section. The killer use case: post a status of `argocd/myapp-prod: success` on the *config repo's commit SHA* (which is the SHA `app.status.sync.revision` references). If the config-repo commit was originally created by a PR from the app repo (with the app-repo SHA in the body), GitHub's cross-repo status linking surfaces "ArgoCD has finished reconciling" as a check on the original feature-branch PR.

2. **Deployments API** -- `POST /repos/{owner}/{repo}/deployments`. Populates the repo's "Environments" sidebar on the home page. After a fresh deploy, the sidebar shows `prod: deployed 4 minutes ago, sha abc1234`. The most-visible, lowest-friction "is it deployed?" view available without leaving GitHub.

3. **Pull request comments** -- rendered as Markdown. Useful for embedding the ArgoCD sync diff or a rollout-health summary directly into the PR conversation. "Deployed to dev: healthy ✅" appearing as a bot comment closes the developer's mental loop in the same UI as the code review.

### GitHub App vs PAT for ArgoCD Notifications

PAT works for testing; GitHub App is the only correct choice for production. Why:

| Property | GitHub App | PAT |
|---|---|---|
| Scope | Per-repo install | User-account-wide (or org via fine-grained) |
| Identity | "ArgoCD Bot" -- always | Tied to a person; breaks when they leave |
| Rotation | Automatic (private key signed JWT) | Manual; humans forget |
| Required scopes | App permissions: `Contents: Read, Pull Requests: Write, Deployments: Write, Statuses: Write` | PAT scopes: `repo:status` + `deployments` + `repo` (broader than App) |
| Rate limits | Higher per-installation | Per-user, shared with everything else the human does |

The App's permissions for full Notifications coverage: **Contents: Read, Pull Requests: Write, Deployments: Write, Statuses: Write**. (Gotcha #11.)

---

## Part 7: The Full Handoff Diagram -- End to End

```
DEVELOPER PUSHES CODE -> PROD DEPLOY: every arrow's owner and auth
==============================================================================

[1] Developer git push origin feat/foo
       Owner: developer's git client
       Auth: SSH key / HTTPS PAT to GitHub

[2] PR opened -> .github/workflows/ci.yml fires
       Owner: GitHub Actions
       Auth: GITHUB_TOKEN scoped to PR's repo
       Action: test/lint/build (NO push to registry yet)

[3] PR merged to main -> .github/workflows/release.yml fires
       Owner: GitHub Actions
       Auth: OIDC -> AssumeRole(BuildPushRole) sub:ref:refs/heads/main
       Action: docker buildx build/push, cosign sign, output digest

[4] release.yml's promote-dev job:
       Owner: GitHub Actions
       Auth: GitHub App token (CONFIG_REPO_APP) -> read+write on config repo
       Action: yq update envs/dev/values.yaml; git commit "[skip ci]"; git push

[5] ArgoCD repo-server polls config repo (default 3 min)
       Owner: ArgoCD
       Auth: ArgoCD's repo creds (separate Secret per repo)
       Action: Detects new commit; computes new manifest; checks Application

[6] ArgoCD application-controller reconciles dev cluster
       Owner: ArgoCD
       Auth: Cluster ServiceAccount (in-cluster) OR cluster Secret for ext clusters
       Action: kubectl apply equivalent; rollout proceeds

[7] ArgoCD Notifications fires on-deployed (Application healthy)
       Owner: ArgoCD Notifications controller
       Auth: GitHub App private key (mounted Secret)
       Action: POST /statuses/{sha}, POST /deployments, POST /pulls/{n}/comments

[8] PR conversation now shows:
       - Original CI check from step [2]: ✅
       - argocd/myapp-dev: ✅ (from step [7])
       - Bot comment: "Deployed to dev: healthy"

[9] (Hours later) Engineering manually triggers Promote dev->stg:
       Either: workflow_dispatch on config repo, OR ArgoCD Image Updater w/ stg
       Owner: GitHub Actions / Image Updater controller
       Auth: GitHub App OR Image Updater's git creds
       Action: Open PR copying dev's digest into envs/stg/values.yaml

[10] Stg PR opens; CODEOWNERS auto-requests SRE review
       Owner: GitHub
       Auth: configured CODEOWNERS rules; branch protection requires 1 approval

[11] SRE approves stg PR; merges
       Owner: human
       Action: triggers same release.yml's promote-stg job (but with environment:stg
                so OIDC sub becomes ...:environment:stg)
                Stg AWS role accepts that sub via trust policy

[12] ArgoCD reconciles stg cluster (steps 5-7 repeat for stg)
       PR comment on the stg-promotion PR: "Deployed to stg: healthy"

[13] Tag-driven prod: developer git tag v1.2.3 && git push --tags
       Owner: developer
       Action: triggers promote-prod.yml on tag event

[14] Prod job declares environment: production
       GitHub gates: required reviewers (2 SREs), wait 30 min, branch:tags-only
       During wait: env-scoped secrets NOT exposed to runner
       sub claim minted ONLY after gates pass: ...:environment:production

[15] After 30-min wait + 2 SRE approvals:
       OIDC AssumeRole(ProdDeployRole) succeeds (trust policy matches sub)
       Job creates prod promotion PR
       Config-repo branch protection requires another 2 SRE approvals to merge

[16] Prod PR merged -> ArgoCD prod (steps 5-7 again)
       prod Application: syncPolicy: { automated: { selfHeal: true, prune: true } }
       Notifications: PR comment "Deployed to prod: healthy"
       GitHub Deployments sidebar: prod = green dot, 2 minutes ago

CARDINAL RULE TEST: count auths.
   16 numbered arrows. 8 distinct auth mechanisms.
   None of the cluster-write creds are ever in CI. None of the Git-write creds
   are ever in the cluster. ArgoCD never opens a PR. CI never `kubectl apply`s.
```

---

## Part 8: 16 Production Gotchas

1. **`peter-evans/create-pull-request` requires a PAT or GitHub App token -- two stacked reasons.** Layer 1 (scope): `GITHUB_TOKEN` is repo-scoped and cannot write to a different repo, so cross-repo PRs simply fail with 403/404. Layer 2 (recursion fuse): even for same-repo writes, events authored by `GITHUB_TOKEN` do not trigger downstream workflows -- a deliberate GitHub anti-recursion fuse, so the auto-PR opens but its CI checks never run and a green merge silently bypasses validation. Use `actions/create-github-app-token` to mint a per-installation App token; it clears both layers and isn't tied to a human's account. Fine-grained PATs work but rotate poorly.

2. **CI auto-commit triggers infinite loop -- and `[skip ci]` alone won't save you on GitHub Actions.** Bot pushes config-repo commit -> config-repo CI runs -> if config-repo CI also rebuilds an artifact -> recursion. **GitHub Actions does NOT honor `[skip ci]` natively** (unlike GitLab CI / CircleCI / Jenkins). Real fixes: **(1) primary**: scope config-repo CI to `paths-ignore: ['apps/**/values.yaml', 'apps/**/version.yaml']` so bot commits to values files never trigger CI. **(2) defense in depth**: bot-author guard `if: github.actor != 'argocd-image-updater[bot]' && github.actor != 'deploy-bot[bot]'`. **(3) explicit message guard if you must**: `if: "!contains(github.event.head_commit.message, '[skip ci]')"` on every job. Keep `[skip ci]` in commit messages as a cross-tool convention, but never rely on GHA respecting it on its own.

3. **ArgoCD Image Updater `argocd` write-back drifts when humans also edit `values.yaml`.** Default write-back method writes parameter overrides via the ArgoCD API into the live Application. The override is invisible from `git diff`; deleting the Application wipes it. *Always* set `write-back-method: git` on Image-Updater-managed Applications. The `argocd` method exists for testing only.

4. **Image Updater branch tracking -- `git-branch` annotation must match Application's `targetRevision`.** If the Application tracks `main` but `git-branch` is `image-bumps`, write-back commits to `image-bumps` (which ArgoCD never reads), and your tag bumps go into a parallel branch nobody syncs. Silent failure. No error. Validate: `kubectl get app -o yaml | grep -E 'targetRevision|git-branch'` and confirm they match.

5. **Helm chart values have BOTH `image.tag` and `image.digest` -- update only one or you get a conflict.** A chart that templates `{{ .Values.image.repository }}:{{ .Values.image.tag }}@{{ .Values.image.digest }}` produces `myapp:v1.2.3@sha256:abc...`, where the digest pins but the tag is just metadata. If your CI updates *both* and they reference different builds, you get `ImagePullBackOff: digest mismatch`. **Pick digest pinning** (the digest wins; leave the tag empty or set it to the immutable image SHA), or pick tag pinning (don't set digest). Don't set both unless your chart explicitly handles the precedence.

6. **GitHub Environments required reviewers can self-approve unless "Prevent self-review" is enabled.** A reviewer who triggered the deploy can approve their own deploy. The toggle was added in late 2024; **enable it on every production environment**. Without it, the reviewer count is ceremony, not enforcement.

7. **Wait timer doesn't pause GHA billing minutes for the wait itself, but the gate-job is billable while waiting in some matrix patterns.** The wait phase doesn't burn minutes. However, if your matrix has `needs: build` and the gating job is the matrix's terminal phase, edge cases (especially with `if:` expressions evaluated mid-wait) can cause stale runners to hold workspaces. Test before relying on a 24-hour wait timer for cost-savings claims.

8. **Deployment branch policy + tag policy don't compose with simple AND semantics.** Both kinds together act as "this deploy must match the *type* that fired it." If you configure branch policy = `main` and tag policy = `v*`, a push to `main` succeeds (branch policy match), a tag push of `v1.2.3` succeeds (tag policy match), but a deploy from `release/foo` fails (no branch match) even though tag policy might "look" permissive. Read the policy as an allowlist per kind, not as a compound predicate.

9. **Environment-scoped secret on a job that doesn't declare `environment:` is silently empty.** No error, just `secrets.PROD_DB_URL` evaluates to ``. The job runs, fails on the missing creds, and the engineer spends an hour debugging. Always pair env-scoped secrets with `environment:` on the consuming job; or move them to repo scope if multiple jobs need them.

10. **OIDC `sub` mismatch when promoting via `environment:`.** A trust policy scoped to `repo:org/repo:ref:refs/heads/main` rejects a job that declares `environment: production` -- the `sub` becomes `repo:org/repo:environment:production`, which doesn't match. Use *separate roles* per phase (build = ref-scoped, deploy = environment-scoped) and trust policies that mirror the OIDC reality, not the workflow file structure.

11. **ArgoCD Notifications GitHub service requires the right App permissions.** Common mis-permissioning: the App has `Contents: Write` but missing `Deployments: Write` -- commit statuses post fine, Deployments API fails silently. Required: **Contents: Read, Pull Requests: Write, Deployments: Write, Statuses: Write**. PATs need `repo:status` + `deployments`. Audit the App's per-installation permissions on the config repo specifically.

12. **Two-repo: ArgoCD Application points to config repo, but config repo's git credentials missing on ArgoCD = 401.** When migrating from single-repo to two-repo, engineers often forget to add the config repo's credentials to ArgoCD's Repositories list. The Application shows "Repository not accessible -- 401 Unauthorized" but only in the ArgoCD UI; nothing surfaces in GitHub. Add the credential via `argocd repo add` or a `Secret` with `argocd.argoproj.io/secret-type: repository` label.

13. **Image Updater allowlist: `helm.image-name` must exactly match the Helm values key being updated.** If your chart consumes `app.image.repo` but your annotation says `helm.image-name: image.repository`, Image Updater writes to `image.repository` -- which the chart never reads -- and the digest never bumps. The bug is silent: no log, no error, just the cluster keeps running the old image. Verify with `argocd-image-updater test <image>` and `helm template ./chart --debug | grep image:` before deploying.

14. **Promotion via tag -- forgetting `actions/checkout@v4 with: fetch-depth: 0` means tag history isn't available.** Default shallow clone (depth 1) doesn't include tags. `git describe --tags` fails, `crane digest <repo>:<previous-tag>` fails because the runner's local repo doesn't know the previous tag exists. Always set `fetch-depth: 0` on tag-driven workflows. The cost is a few seconds of git clone time; the alternative is mysterious failures.

15. **Self-hosted runners + OIDC: `id-token: write` is still required.** Even on a self-hosted runner that obtains AWS creds from an instance profile, if you use OIDC for *anything else* in the workflow (e.g., Cosign keyless signing), the workflow still needs `permissions: id-token: write` to mint the OIDC JWT. Symptom: signing fails with `unable to obtain id-token` even though AWS push works fine.

16. **Image Updater / Kargo commits must not trigger CI workflows -- and on GHA, the only reliable filter is `paths-ignore` or a bot-author `if:` guard.** Default Image Updater commit-message template includes no `[skip ci]`, but adding one wouldn't help on GHA anyway (see gotcha #2). Real fixes: **(a) `paths-ignore: ['apps/**/values.yaml']` in the workflow** so bot commits to values files don't trigger CI in the first place. **(b) bot-author guard**: `if: github.actor != 'argocd-image-updater[bot]'`. Override the Image Updater commit-message template if you also mirror to GitLab/Circle (where `[skip ci]` does work): `argocd-image-updater.argoproj.io/git.commit-message-template: 'chore: bump {{.AppName}} to {{.AlleastedTag}} [skip ci]'`.

---

## Part 9: Decision Frameworks

### Framework 1 -- Image-Updater vs hand-rolled-PR vs Kargo

| You have... | Pick | Why |
|---|---|---|
| 1-50 services, app-team owns deploy | Hand-rolled PR | Highest audit fidelity, simplest mental model |
| 1-200 services, platform-team owns config repo | Image Updater | Decouples CI from config repo entirely; CI is registry-only |
| 50+ services, multi-region, SLO-gated promotion | Kargo | First-class Stage graph + verification; UI for pipeline-wide visibility |
| 5-10 services, single region, fast iteration | Hand-rolled or Image Updater | Either works; pick by team's familiarity |
| You want PR-based promotion AND don't want to write workflows | GitOps Promoter | Smaller scope than Kargo; tighter ArgoCD-native PR-promotion |

### Framework 2 -- Two-repo vs single-repo

| Factor | Two-repo | Single-repo |
|---|---|---|
| Team size | 5+ engineers | <5 engineers |
| Permission model | "Engineering writes code; SRE writes deploys" | "Same humans do both" |
| Audit complexity | High (needs separation) | Low (single timeline OK) |
| Multi-tenant ArgoCD | Yes | No |
| Promotion workflow | Cross-repo PR | Same-repo path-scoped PR |

### Framework 3 -- Auto-promote vs PR-promote vs tag-promote per environment

| Environment | Pattern | Reasoning |
|---|---|---|
| `dev` | Auto-promote on push-to-main | Fast feedback; the env explicitly accepts breakage |
| `stg` | PR-promote (CI opens PR, human merges) | Catch dev-stg drift; review CI test results |
| `prod` | Tag-promote (`v*` only) + GitHub Environment with reviewers + 30-min wait | Explicit release ceremony; protection-rule belt-and-suspenders |
| `canary` (optional) | Auto-promote with verification gate (Argo Rollouts AnalysisTemplate) | Fastest signal on real traffic; auto-rollback on failure |

### Framework 4 -- GitHub Environments vs ArgoCD AppProject vs Kargo Stage for gating

| Layer | Threat it mitigates | When required |
|---|---|---|
| GitHub Environments | Malicious / careless PR triggers prod deploy | Always for prod |
| ArgoCD AppProject | Compromised ArgoCD deploys arbitrary workloads to prod | Always (multi-tenant clusters) |
| Kargo Stage | Promotion bypasses verification (canary, SLO) | When verification is a hard requirement |

**Most teams need both Environments and AppProject. Kargo Stage adds a third layer when verification matters.**

### Framework 5 -- GITHUB_TOKEN vs PAT vs GitHub App for cross-repo writes

| Use case | Pick | Why |
|---|---|---|
| Same-repo writes | GITHUB_TOKEN | Default; minimal blast radius |
| Cross-repo writes; downstream CI must trigger | GitHub App | Triggers workflows; auto-rotates; per-repo install |
| Cross-repo writes; no downstream CI; quick prototype | Fine-grained PAT | Easiest setup; tied to a person; bad for production |
| Cross-org writes | GitHub App with multi-org install | Per-org installation; explicit consent model |

---

## Part 10: Production Examples (Full Runnable Code)

### Example 1: Two-repo `release.yml` -- build, sign, commit-to-config-repo

```yaml
# In app-repo: .github/workflows/release.yml
name: Release
on:
  push:
    branches: [main]

permissions:
  contents: read

env:
  AWS_REGION: us-east-1
  IMAGE_REPO: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
      packages: write
    outputs:
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v5
      - uses: docker/setup-qemu-action@v4
      - uses: docker/setup-buildx-action@v4
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GHA-MyApp-BuildPushRole
          aws-region: ${{ env.AWS_REGION }}
          role-session-name: GHA-build-${{ github.run_id }}-${{ github.run_attempt }}
      - id: ecr-login
        uses: aws-actions/amazon-ecr-login@v2
      - id: meta
        uses: docker/metadata-action@v6
        with:
          images: ${{ env.IMAGE_REPO }}
          tags: |
            type=sha,format=long
            type=ref,event=branch
            type=raw,value=latest,enable={{is_default_branch}}
      - id: build
        uses: docker/build-push-action@v7
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha,scope=myapp
          cache-to: type=gha,mode=max,scope=myapp
          provenance: mode=max
          sbom: true
      - uses: sigstore/cosign-installer@v4
      - name: Cosign sign by digest
        run: |
          cosign sign --yes ${{ env.IMAGE_REPO }}@${{ steps.build.outputs.digest }}

  promote-dev:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: dev
      url: https://argocd.example.com/applications/myapp-dev
    permissions:
      contents: read
    steps:
      - name: Mint config-repo App token
        id: token
        uses: actions/create-github-app-token@v2
        with:
          app-id: ${{ vars.CONFIG_REPO_APP_ID }}
          private-key: ${{ secrets.CONFIG_REPO_APP_PRIVATE_KEY }}
          owner: ${{ github.repository_owner }}
          repositories: my-config-repo

      - uses: actions/checkout@v5
        with:
          repository: ${{ github.repository_owner }}/my-config-repo
          token: ${{ steps.token.outputs.token }}
          path: cfg
          fetch-depth: 0

      - name: Bump dev digest
        working-directory: cfg
        env:
          NEW_DIGEST: ${{ needs.build.outputs.digest }}
        run: |
          yq -i '.image.digest = strenv(NEW_DIGEST)' apps/myapp/envs/dev/values.yaml
          git config user.email "deploy-bot@my-org.com"
          git config user.name "Deploy Bot"
          git add apps/myapp/envs/dev/values.yaml
          git commit -m "chore(myapp): bump dev to ${NEW_DIGEST} [skip ci]"
          git push origin main

  open-stg-promotion-pr:
    needs: promote-dev
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - name: Mint App token
        id: token
        uses: actions/create-github-app-token@v2
        with:
          app-id: ${{ vars.CONFIG_REPO_APP_ID }}
          private-key: ${{ secrets.CONFIG_REPO_APP_PRIVATE_KEY }}
          owner: ${{ github.repository_owner }}
          repositories: my-config-repo

      - uses: actions/checkout@v5
        with:
          repository: ${{ github.repository_owner }}/my-config-repo
          token: ${{ steps.token.outputs.token }}
          path: cfg

      - name: Update stg values
        working-directory: cfg
        env:
          NEW_DIGEST: ${{ needs.build.outputs.digest }}
        run: |
          yq -i '.image.digest = strenv(NEW_DIGEST)' apps/myapp/envs/stg/values.yaml

      - uses: peter-evans/create-pull-request@v7
        with:
          path: cfg
          token: ${{ steps.token.outputs.token }}
          branch: promote/myapp-stg-${{ github.run_id }}
          delete-branch: true
          commit-message: "chore(myapp): promote dev->stg ${{ needs.build.outputs.digest }} [skip ci]"
          title: "Promote myapp dev->stg"
          body: |
            Promote dev's image digest to stg.
            - Digest: `${{ needs.build.outputs.digest }}`
            - From app repo SHA: `${{ github.sha }}`
            - Workflow: ${{ github.run_id }}
          signoff: true
          labels: env:stg, promotion
```

### Example 2: ArgoCD Application with Image Updater annotations

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-dev
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: myapp=123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp
    argocd-image-updater.argoproj.io/myapp.update-strategy: digest
    argocd-image-updater.argoproj.io/myapp.allow-tags: regexp:^main$
    argocd-image-updater.argoproj.io/myapp.helm.image-name: image.repository
    # See Part 2 note: in digest mode, `helm.image-tag` is the slot name not a
    # type assertion. Cleaner alternatives: `helm.image-spec: image.fqin` (single
    # combined field) or `manifestTargets.helm.digest` (v0.13+, three separate keys).
    argocd-image-updater.argoproj.io/myapp.helm.image-tag: image.digest
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/write-back-target: helmvalues:./apps/myapp/envs/dev/values.yaml
    argocd-image-updater.argoproj.io/git-branch: main
    argocd-image-updater.argoproj.io/git.commit-message-template: |
      chore(myapp): {{ range .AppChanges -}}
      {{ .Image }}: {{ .OldTag }} -> {{ .NewTag }}
      {{ end }}[skip ci]
    argocd-image-updater.argoproj.io/myapp.pull-secret: ext:/scripts/auth1.sh
    notifications.argoproj.io/subscribe.on-deployed.github-deployed: ""
    notifications.argoproj.io/subscribe.on-health-degraded.github-degraded: ""
    notifications.argoproj.io/subscribe.on-sync-failed.github-sync-failed: ""
  labels:
    env: dev
spec:
  project: myapp
  source:
    repoURL: https://github.com/my-org/my-config-repo.git
    targetRevision: main
    path: apps/myapp/envs/dev
    helm:
      valueFiles: [values.yaml]
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Example 3: workflow_dispatch with environment input

```yaml
name: Manual deploy
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options: [dev, stg, prod]
      digest:
        description: 'Image digest (sha256:...) - copy from successful release run'
        required: true
        type: string

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    permissions:
      id-token: write
      contents: write
    steps:
      - uses: actions/checkout@v5

      - name: Validate digest format
        run: |
          if [[ ! "${{ inputs.digest }}" =~ ^sha256:[a-f0-9]{64}$ ]]; then
            echo "::error::Invalid digest format"
            exit 1
          fi

      - name: Update values
        env:
          DIGEST: ${{ inputs.digest }}
          ENV: ${{ inputs.environment }}
        run: |
          yq -i '.image.digest = strenv(DIGEST)' "apps/myapp/envs/${ENV}/values.yaml"
          git config user.email deploy-bot@my-org.com
          git config user.name "Deploy Bot"
          git commit -am "chore(myapp): manual deploy ${DIGEST} to ${ENV} [skip ci]"
          git push
```

### Example 4: ArgoCD Notifications ConfigMap (production)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-notifications-secret
  namespace: argocd
stringData:
  github-appid: "123456"
  github-installid: "789012"
  github-privatekey: |
    -----BEGIN RSA PRIVATE KEY-----
    ...
    -----END RSA PRIVATE KEY-----
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.github: |
    appID: $github-appid
    installationID: $github-installid
    privateKey: $github-privatekey
  context: |
    argocdUrl: https://argocd.example.com
  trigger.on-deployed: |
    - description: Application synced and healthy
      send: [github-deployed]
      when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
      oncePer: app.status.sync.revision
  trigger.on-health-degraded: |
    - description: Application degraded
      send: [github-degraded]
      when: app.status.health.status == 'Degraded'
  trigger.on-sync-failed: |
    - description: Sync operation failed
      send: [github-sync-failed]
      when: app.status.operationState.phase in ['Error', 'Failed']
  template.github-deployed: |
    message: ":white_check_mark: {{.app.metadata.name}} deployed @ {{.app.status.sync.revision}}"
    github:
      status:
        state: success
        label: "argocd/{{.app.metadata.name}}"
        targetURL: "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}"
      deployment:
        state: success
        environment: "{{.app.metadata.labels.env}}"
        environmentURL: "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}"
        logURL: "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}?operation=true"
        requiredContexts: []
        autoMerge: true
  template.github-degraded: |
    message: ":x: {{.app.metadata.name}} health degraded"
    github:
      status:
        state: failure
        label: "argocd/{{.app.metadata.name}}"
        targetURL: "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}"
  template.github-sync-failed: |
    message: ":x: {{.app.metadata.name}} sync failed: {{.app.status.operationState.message}}"
    github:
      status:
        state: failure
        label: "argocd/{{.app.metadata.name}}"
        targetURL: "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}"
```

### Example 5: Terraform for GitHub Environments + per-env IAM trust policies

```hcl
locals {
  github_org  = "my-org"
  github_repo = "my-app"
  envs = {
    dev  = { reviewers = [], wait_minutes = 0,  branches = ["main"] }
    stg  = { reviewers = ["sre"], wait_minutes = 0, branches = ["main"] }
    prod = { reviewers = ["sre", "eng-lead"], wait_minutes = 30, tags = ["v*"], branches = [] }
  }
}

# GitHub provider
data "github_team" "reviewers" {
  for_each = toset(flatten([for env in local.envs : env.reviewers]))
  slug     = each.key
}

resource "github_repository_environment" "env" {
  for_each    = local.envs
  repository  = local.github_repo
  environment = each.key
  prevent_self_review = each.key == "prod" ? true : false

  dynamic "reviewers" {
    for_each = length(each.value.reviewers) > 0 ? [1] : []
    content {
      teams = [for t in each.value.reviewers : data.github_team.reviewers[t].id]
    }
  }

  wait_timer = each.value.wait_minutes

  deployment_branch_policy {
    protected_branches     = false
    custom_branch_policies = true
  }
}

resource "github_repository_environment_deployment_policy" "branches" {
  for_each       = { for k, v in local.envs : k => v if length(v.branches) > 0 }
  repository     = local.github_repo
  environment    = github_repository_environment.env[each.key].environment
  branch_pattern = each.value.branches[0]
}

# AWS provider -- one role per env, scoped trust policy
data "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"
}

resource "aws_iam_role" "deploy_per_env" {
  for_each = local.envs
  name     = "GHA-${local.github_repo}-Deploy-${each.key}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = data.aws_iam_openid_connect_provider.github.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
          # KEY: env-scoped sub binds the role to GitHub's protection-rule enforcement.
          "token.actions.githubusercontent.com:sub" = "repo:${local.github_org}/${local.github_repo}:environment:${each.key}"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy" "deploy_perms" {
  for_each = local.envs
  role     = aws_iam_role.deploy_per_env[each.key].id
  policy   = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      # Minimum-necessary -- scoped per env's needs (e.g., only prod role can publish to prod's CodeDeploy)
      Action   = ["s3:PutObject", "s3:GetObject"]
      Resource = "arn:aws:s3:::myapp-${each.key}-deploy-artifacts/*"
    }]
  })
}

# Variables that the workflow consumes
resource "github_actions_environment_variable" "deploy_role_arn" {
  for_each      = local.envs
  repository    = local.github_repo
  environment   = github_repository_environment.env[each.key].environment
  variable_name = "DEPLOY_ROLE_ARN"
  value         = aws_iam_role.deploy_per_env[each.key].arn
}
```

---

## Part 11: Cheat Sheet

### Three image-update strategies

| Strategy | Trigger | Writes to | Use when |
|---|---|---|---|
| CI commits PR | Push to main / tag | Config repo via `peter-evans/create-pull-request` + GitHub App | Default; team owns deploy |
| Image Updater | Registry poll | Config repo via `write-back-method: git` | Decoupled CI; platform team owns config |
| Kargo / Promoter | Stage CRD reconciliation | Config repo + ArgoCD via promotion steps | Multi-env, verification gates |

### Four promotion patterns

| Pattern | Trigger | Used for |
|---|---|---|
| Auto-promote | Push to main | dev only |
| PR-promote | Auto-PR by CI; human merge | stg |
| Tag-promote | Push tag `v*` | prod |
| `workflow_dispatch` | Manual operator trigger | All envs (re-deploy, hotfix) |

### GitHub Environments protection rules

| Rule | Max | Composes with | Cost on Free private |
|---|---|---|---|
| Required reviewers | 6 users/teams | All others | Pro+ only |
| Wait timer | 30 days | All others | Pro+ only |
| Branch policy | Allowlist + protected | Tag policy via OR-per-kind | Pro+ only |
| Tag policy | Allowlist (2024+) | Branch policy via OR-per-kind | Pro+ only |
| Custom rule | One App per env | Optional add-on | **Enterprise only** (public repos: all plans) |
| Prevent self-review | Toggle | Required reviewers | Free |

### OIDC `sub` claim formats by event

| Event | `sub` |
|---|---|
| Push to main | `repo:org/repo:ref:refs/heads/main` |
| Push to tag | `repo:org/repo:ref:refs/tags/v1.2.3` |
| Pull request | `repo:org/repo:pull_request` (DANGEROUS) |
| Job with `environment: production` | `repo:org/repo:environment:production` |
| Reusable workflow `job_workflow_ref` claim | `org/shared-workflows/.github/workflows/x.yml@refs/heads/main` |

### ArgoCD Notifications built-in triggers

| Trigger | Fires on |
|---|---|
| `on-deployed` | Application synced + healthy |
| `on-health-degraded` | Application healthy → degraded |
| `on-sync-failed` | Sync operation Error/Failed |
| `on-sync-status-unknown` | Application sync status Unknown |
| `on-sync-running` | Sync started |
| `on-sync-succeeded` | Sync succeeded (regardless of health) |
| `on-created` | Application CR created |
| `on-deleted` | Application CR deleted |

---

## The 10 Commandments of GitOps CD

1. **Thou shalt not `kubectl apply` from CI.** The cluster pulls; CI pushes only to Git.
2. **Thou shalt commit digests, not tags.** Tags lie; digests are the bytes you signed.
3. **Thou shalt scope OIDC `sub` to the environment, not the branch, for prod.** `environment:production` composes with required reviewers; `ref:refs/heads/main` does not.
4. **Thou shalt use a GitHub App for cross-repo writes.** PATs leak; `GITHUB_TOKEN` doesn't trigger downstream workflows; Apps trigger workflows AND auto-rotate.
5. **Thou shalt fence the recursion loop with `paths-ignore` or bot-author guards.** GitHub Actions does NOT honor `[skip ci]` natively -- that's a GitLab/Circle convention. Mark bot commits with `[skip ci]` for cross-tool hygiene, but the actual fence on GHA is path filters and `github.actor` checks.
6. **Thou shalt use `git` write-back, never `argocd`, for Image Updater.** In-cluster overrides drift from Git; commits don't.
7. **Thou shalt enforce folders, not branches, per environment.** `git merge` from dev to prod is a war crime; `cp version.yaml` is a promotion.
8. **Thou shalt enable Prevent Self-Review on prod environments.** Required reviewers without it is ceremony.
9. **Thou shalt close the loop with ArgoCD Notifications.** A green CI check is "manifest written"; a green status from the cluster is "running healthy."
10. **Thou shalt pick exactly one image-update mechanism per repo.** Two automations on the same `values.yaml` is a war between two clerks scribbling over each other.

---

## Further Reading

- OpenGitOps spec (CNCF): <https://opengitops.dev/>
- Kostis Kapelonis, "How to Model Your GitOps Environments": <https://octopus.com/blog/how-to-model-your-gitops-environments>
- ArgoCD Image Updater -- update methods: <https://argocd-image-updater.readthedocs.io/en/stable/basics/update-methods/>
- ArgoCD Notifications GitHub service: <https://argo-cd.readthedocs.io/en/stable/operator-manual/notifications/services/github/>
- GitHub Environments: <https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments>
- `peter-evans/create-pull-request`: <https://github.com/peter-evans/create-pull-request>
- Kargo (Akuity): <https://akuity.io/blog/promotion-made-easy-with-kargo>
- GitOps Promoter (argoproj-labs): <https://github.com/argoproj-labs/gitops-promoter>
- May 7 doc -- container CI build/scan/sign/push: `2026/may/cicd/2026-05-07-github-actions-container-ci-pipeline.md`
- April 19 doc -- ArgoCD + Helm production patterns: `2026/april/helm-and-argocd/2026-04-19-argocd-helm-production-patterns.md`
