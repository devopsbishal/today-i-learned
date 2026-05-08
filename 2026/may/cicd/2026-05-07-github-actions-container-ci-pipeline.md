# GitHub Actions Container CI -- Build, Scan, Sign & Push -- The Bakery Pickup Window with the Tamper-Evident Seal

> The last two days were about the *kitchen layout*: how runners spin up, how jobs flow, how composites and reusable workflows compose YAML reuse. That covered "how the building works." This doc is about the *production line that ships a finished product to the loading dock* -- the specific assembly line every container CI pipeline becomes once you stop hand-rolling `docker build && docker push` and start treating images as **signed, scanned, attested supply-chain artifacts** that downstream Kubernetes admission controllers will refuse to run unless every seal is intact. This is where six previously-separate concepts -- BuildKit, registry auth, vulnerability scanning, SBOM generation, code signing, OIDC federation -- collapse into a single linear `release.yml` of about 120 lines, and where one wrong character in an IAM trust policy quietly opens your AWS account to every fork's pull request.
>
> **The core analogy for today: a bakery's pickup window where every loaf gets a tamper-evident holographic seal before it leaves the building.** Walk it through end-to-end. The flour-sack delivery dock is `docker/setup-qemu-action` (the side gate that lets in *non-native* flour -- arm64 wheat for an amd64 oven). The mise-en-place setup is `docker/setup-buildx-action` (the bench gets a calibrated scale, the BuildKit oven gets pre-heated, the GHA cache backend gets pre-wired). The recipe slip with the day's tags is `docker/metadata-action` (today's sourdough is labeled `v1.4.2`, `v1.4`, `v1`, `latest`, `sha-abc1234` -- five labels for the same loaf, deterministic, never invented at runtime). The actual baking is `docker/build-push-action` (BuildKit's oven, multi-stage, layer-cached). The X-ray scanner at the back of the kitchen is `aquasecurity/trivy-action` (every loaf passes through; if the scan finds a glass shard above the team's tolerance, the loaf is binned and the line stops). The receipt that lists every grain of flour and the supplier's lot number is the **BuildKit SBOM attestation** (in-toto JSON, attached to the loaf's wrapping, travels with it forever). The holographic tamper-evident seal is **Cosign keyless signing** (Fulcio prints a 10-minute disposable certificate that says "this exact loaf, baked by THIS bakery on THIS shift, at this digest"; the seal is registered in Rekor, the public transparency log, so anyone can later prove it wasn't forged). The courier picking the loaf up doesn't show their personal driver's license -- they swipe a **per-shift OIDC badge** (`token.actions.githubusercontent.com`) that the AWS receiving dock has pre-authorized via the **IAM identity provider + trust policy** (the dock supervisor's clipboard says "loaves from `repo:my-org/my-repo:ref:refs/heads/main` are allowed; everything else turn away"). The walk-in cooler at the receiving dock is **ECR**. **One bakery, one shift, one loaf, five tags, four attestations, zero long-lived keys.**
>
> Three corollaries this analogy carries that show up in every section below. **(1) The seal is on the loaf, not on the label.** The single most common authoring mistake in container CI is signing a *tag* (`my-app:v1.2.3`) instead of a *digest* (`my-app@sha256:abc...`). Tags are mutable -- you can re-point `v1.2.3` at a different image tomorrow -- so signing a tag is signing a sticker that someone can later peel off and stick on a different loaf. The digest is the cryptographic identity of the bytes; sign that. `docker/build-push-action@v7` exposes `steps.<id>.outputs.digest` precisely so you can pipe it straight into `cosign sign`. Every time you read "verify by digest" in this doc, hear the echo of "the seal is on the loaf, not on the label." **(2) The badge is per-shift, not per-employee.** OIDC federation replaces long-lived `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` GitHub secrets (which are like handing every kitchen employee a permanent master key to the dock) with a per-workflow-run JWT minted on the fly by GitHub, redeemable for STS credentials whose lifetime is measured in minutes. The lifecycle: `permissions: id-token: write` is "give the courier today's badge"; the IAM trust policy's `sub` claim is "the dock supervisor's clipboard"; `aws-actions/configure-aws-credentials@v6` is "the courier shows the badge, the dock checks the clipboard, hands over a temporary clearance card good for one hour." Get the `sub` wrong and either the courier is rejected (`Could not load credentials`) or, much worse, the door is wedged open for any courier from any bakery (`sub: repo:my-org/my-repo:*` accepts any workflow on any ref, including pull requests from forks). **(3) Cache scope is per-recipe, not per-bakery.** GitHub's Actions cache is a 10 GB shared larder per repo. If you bake four different breads and don't `scope=` them apart, each bread's BuildKit layer cache evicts the previous one's, and your "cache-enabled" pipeline silently runs cold every time. Scope per image (`scope=api`, `scope=worker`, `scope=migrator`) and the larder is partitioned correctly. This is the single most-overlooked tuning dial in monorepo container CI.

---

**Date**: 2026-05-07
**Topic Area**: cicd
**Difficulty**: Intermediate -> Advanced

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| The four-action stack | Flour delivery -> bench setup -> oven -> seal-and-pack | `setup-qemu` + `setup-buildx` + `login` + `build-push` -- memorize as one block |
| `docker/setup-qemu-action@v4` | The side gate for non-native flour | Multi-arch emulation; only needed when building `linux/arm64` from amd64 runners |
| `docker/setup-buildx-action@v4` | Pre-heating the BuildKit oven; calibrating the bench | Boots a BuildKit builder; auto-wires the GHA cache v2 backend |
| `docker/login-action@v4` | The courier badge swipe at the dock | Registry auth; takes registry + username + password (or temp creds from ECR action) |
| `docker/build-push-action@v7` | The actual baking | Builds + tags + pushes; emits `digest` and `metadata` outputs |
| `docker/metadata-action@v6` | Today's recipe slip with all five labels | Generates `tags:` and `labels:` deterministically from event context |
| OCI labels | The tamper-evident sticker that says "baked by..." | `org.opencontainers.image.source` makes "View source" work on registry UIs |
| `type=gha` cache | The bakery's shared 10 GB larder | BuildKit cache stored in GitHub Actions cache; per-repo, per-scope |
| `type=registry` cache | A second walk-in across town for the bulk pantry | BuildKit cache pushed to a registry tag; unlimited, cross-fork sharing |
| `type=inline` cache | Stuffing the recipe under the loaf's wrapper | Cache embedded in the image manifest; only `mode=min`, bloats the image |
| `mode=max` vs `mode=min` | "Save every prep stage" vs "Save only the finished plating" | `max` caches all layers including intermediate builder stages |
| Cache `scope=` | Labeling the larder shelf by recipe | Per-image scoping for monorepos; without this, breads evict each other |
| Cache immutability | "The labeled prep container is sealed forever" | A cache key written once cannot be overwritten; bump segments to refresh |
| OIDC federation | Per-shift courier badges instead of permanent master keys | GitHub mints a JWT; AWS exchanges for STS creds; no long-lived secrets |
| `permissions: id-token: write` | "Authorize the courier to draw today's badge" | Without this, OIDC token is not minted; classic foot-gun |
| IAM identity provider | The dock supervisor's pre-approved courier service | One-time per AWS account; trusts `token.actions.githubusercontent.com` |
| Trust policy `sub` claim | The supervisor's clipboard of allowed couriers | The line that decides which workflow can assume the role |
| `repo:org/repo:ref:refs/heads/main` | "Only the main-branch shift" | Branch-scoped; the safe production default |
| `repo:org/repo:environment:prod` | "Only when the dispatcher signs off" | Environment-scoped; combines with required reviewers |
| `repo:org/repo:pull_request` | The wide-open dock door | DANGEROUS -- same `sub` for every PR, including from forks |
| `repo:org/repo:*` wildcard | "Anyone with a clipboard, walk on in" | DOUBLE DANGEROUS -- any workflow on any ref |
| `aws-actions/configure-aws-credentials@v6` | The badge-check at the dock | Calls `sts:AssumeRoleWithWebIdentity`; exports temp creds as env vars |
| `role-session-name` | The courier's signature on the receipt | Should include `${{ github.run_id }}` for CloudTrail attribution |
| Session tagging | Stamps on the receipt for the audit clerk | Passes `repository`, `actor` etc. as STS tags for IAM `Condition` keys |
| `aws-actions/amazon-ecr-login@v2` | "ECR specifically, not generic AWS" | Consumes temp creds; emits `registry` output for `docker/login-action` |
| Trivy hard gate | The X-ray scanner that stops the line | `severity: CRITICAL,HIGH` + `exit-code: 1` -- block the build |
| Trivy two-tier gate | Stop on glass shards; log everything else | Two invocations: SARIF upload for all, hard fail only on CRITICAL/HIGH |
| `.trivyignore` | The accepted-risk register on the supervisor's wall | Allowlist of CVE IDs the team has reviewed; include comments + dates |
| SARIF upload | Filing the X-ray report in the company log | `github/codeql-action/upload-sarif@v4` (NOT v3); annotates PR diffs |
| BuildKit attestations | The supplier-lot receipt stapled to the loaf wrapper | `provenance: true` + `sbom: true` produce in-toto JSON in the manifest |
| In-toto media type | The standardized receipt format | `application/vnd.in-toto+json` -- registries that don't support it strip it |
| Cosign keyless signing | The holographic tamper-evident seal | OIDC -> Fulcio cert (10 min) -> sign digest -> push sig -> Rekor log |
| Fulcio | The notary that prints the disposable certificate | Sigstore's CA; binds workflow identity to ephemeral signing key |
| Rekor | The public ledger book of every seal ever issued | Transparency log; verifiable independently |
| `--certificate-identity-regexp` | The receiving dock's "approved bakeries" list | Policy gate at admission; anchor with `^...$` to prevent prefix matches |
| Sign by digest, never tag | "The seal goes on the loaf, not on the sticker" | Tags are mutable; digests are the bytes; always sign `@sha256:...` |

---

## The Whole Picture in One Diagram

```
THE BAKERY PICKUP WINDOW: BUILD -> SCAN -> SIGN -> PUSH
==============================================================================

GITHUB ACTIONS RUNNER (the bakery)                        AWS / REGISTRY (the dock)

+--------------------------------------------------+
| permissions:                                     |
|   id-token: write    <-- "draw today's badge"    |
|   contents: read                                 |
|                                                  |
| 1. actions/checkout                              |
|       (source dough: Dockerfile + app code)      |
|                                                  |
| 2. docker/setup-qemu-action@v4                   |
|       (side gate: arm64 emulation if needed)     |
|                                                  |
| 3. docker/setup-buildx-action@v4                 |
|       (preheat oven, wire GHA cache v2)          |
|                                                  |
| 4. docker/metadata-action@v6                     |
|       (recipe slip: 5 tags + OCI labels)         |
|       outputs: tags, labels                      |
|                                                  |
| 5. aws-actions/configure-aws-credentials@v6      |     OIDC JWT  +-----------------------+
|       OIDC handshake ----------------------------|---------------> sts:AssumeRoleWith    |
|       role-to-assume: arn:aws:iam:...:role/X     |                 WebIdentity           |
|       outputs: temp AWS_ACCESS_KEY_ID etc.       |     temp creds  IAM identity provider |
|                                                  |<----------------+ trust policy `sub`  |
| 6. aws-actions/amazon-ecr-login@v2               |                 +-----------------------+
|       outputs: registry (123.dkr.ecr...)         |
|       outputs: docker_username + password        |
|                                                  |
| 7. docker/build-push-action@v7                   |     push +-----------------+
|       tags: <from step 4>                        |---------> ECR / GHCR     |
|       labels: <from step 4>                      |          + image manifest |
|       cache-from: type=gha,scope=api             |          + SBOM attest    |
|       cache-to:   type=gha,mode=max,scope=api    |          + provenance     |
|       provenance: true                           |          +-----------------+
|       sbom:       true                           |
|       outputs: digest = sha256:abc123...         |
|                                                  |
| 8. aquasecurity/trivy-action@v0.36               |
|       scanners: vuln,secret,misconfig            |
|       severity: CRITICAL,HIGH                    |
|       exit-code: 1   <-- STOP THE LINE on CVE    |
|       format: sarif (also)                       |
|                                                  |
|    github/codeql-action/upload-sarif@v4          |     +-----------------+
|       (NOT v3 -- v4 is current)                  |---->| Code Scanning   |
|                                                  |     | (Security tab)  |
|                                                  |     +-----------------+
|                                                  |
| 9. sigstore/cosign-installer@v4                  |     +-----------------+
|    cosign sign <ECR>/img@${{ digest }}           |---->| Fulcio (cert)   |
|       (uses GitHub OIDC token; 10-min cert       |     | Rekor (log)     |
|        binds workflow identity to signing key)   |     | ECR (.sig blob) |
|                                                  |     +-----------------+
|                                                  |
+--------------------------------------------------+
       outputs.digest -> downstream GitOps update
                         (bump values.yaml, ArgoCD reconciles)

ADMISSION CONTROLLER (Kyverno / Sigstore Policy Controller) AT POD STARTUP
==============================================================================
   cosign verify <image>@<digest> \
     --certificate-identity-regexp '^https://github.com/my-org/[^/]+/.+$' \
     --certificate-oidc-issuer 'https://token.actions.githubusercontent.com'
   --> If signature missing OR identity doesn't match, Pod is REJECTED.
```

---

## Part 1: The Four-Action Stack -- Memorize This Skeleton

If you remember nothing else from this doc, pin these four actions to memory as a single block. **Every container CI pipeline you ever write starts with these four steps in this order.**

```yaml
- name: Set up QEMU                                     # 1. arm64 emulation gate
  uses: docker/setup-qemu-action@v4
  # Skip if you only build linux/amd64 on amd64 runners

- name: Set up Docker Buildx                            # 2. Pre-heat BuildKit
  uses: docker/setup-buildx-action@v4
  # Boots a BuildKit instance; auto-configures GHA cache v2 backend
  # (sets `url` and `token` parameters automatically since GHA cache v1 EOL'd)

- name: Login to ECR                                    # 3. Registry auth
  uses: docker/login-action@v4
  with:
    registry: ${{ steps.ecr-login.outputs.registry }}   # 123.dkr.ecr.us-east-1.amazonaws.com
    username: ${{ steps.ecr-login.outputs.docker_username_... }}
    password: ${{ steps.ecr-login.outputs.docker_password_... }}

- name: Build and push                                  # 4. Bake + ship
  id: build
  uses: docker/build-push-action@v7
  with:
    context: .
    file: ./Dockerfile
    push: true
    tags: ${{ steps.meta.outputs.tags }}
    labels: ${{ steps.meta.outputs.labels }}
    cache-from: type=gha,scope=${{ env.IMAGE_NAME }}
    cache-to: type=gha,mode=max,scope=${{ env.IMAGE_NAME }}
    provenance: true
    sbom: true
    platforms: linux/amd64
```

That's the skeleton. Everything else in this doc is a story about what gets bolted onto it: deterministic tag generation (Part 2), cache scoping (Part 3), OIDC for ECR auth (Part 4-5), Trivy as a gate between build and push verification (Part 6), BuildKit attestations as the receipt (Part 7), Cosign as the seal (Part 8), and a flagship `release.yml` that wires it all (Part 10).

### Why this exact order

The action ordering is not stylistic -- it's causal. **QEMU before buildx** because `setup-buildx-action` registers QEMU emulators with BuildKit at boot; if QEMU isn't installed yet, multi-arch builds silently fall back to native-only. **Buildx before login** because the GHA cache backend wires up at `setup-buildx-action` time, and if you skip buildx and just use the legacy `docker build` shipped on the runner, you get *zero* layer caching across runs. **Login before build-push** because `docker/build-push-action` calls `docker push` under the hood and will fail with `denied: requested access to the resource is denied` if the daemon isn't authenticated. **Metadata before build** (we'll meet metadata in Part 2) because `build-push-action` consumes `tags:` and `labels:` as inputs.

### The version table (verified 2026-05-07)

| Action | Current major | Latest patch | Notes |
| --- | --- | --- | --- |
| `docker/setup-qemu-action` | **v4** | v4.0.0 | Major bumped Mar 2026 |
| `docker/setup-buildx-action` | **v4** | v4.0.0 | Major bumped Mar 2026 |
| `docker/login-action` | **v4** | v4.1.0 | Major bumped Mar 2026 |
| `docker/metadata-action` | **v6** | v6.0.0 | Major bumped Mar 2026 |
| `docker/build-push-action` | **v7** | v7.1.0 | Major bumped Mar 2026 |
| `aws-actions/configure-aws-credentials` | **v6** | v6.1.1 | Major bumped Feb 2026 |
| `aws-actions/amazon-ecr-login` | **v2** | v2.1.5 | v2 still current |
| `aquasecurity/trivy-action` | v0.x | v0.36.0 | Pre-1.0; pin to specific tag |
| `sigstore/cosign-installer` | **v4** | v4.1.2 | Major bumped 2026; v3 EOL'd Node 16 |
| `github/codeql-action/upload-sarif` | **v4** | v4.35.3 | v4 is current; v3 still parallel-patched |
| `actions/attest-build-provenance` | **v4** | v4.1.0 | GitHub-native attestation alternative |

For production, SHA-pin every third-party action (yesterday's Tj-actions / CVE-2025-30066 lesson applies in full force here -- a compromised version of `aws-actions/configure-aws-credentials` would mint short-lived but production-blast-radius STS creds for the attacker). The major-version float (`@v6`) is fine while learning, but flip to SHA-pinning the moment you go to a real organization.

---

## Part 2: `docker/metadata-action@v6` -- The Recipe Slip

The naive way to tag an image is to write 25 lines of bash:

```yaml
# DON'T DO THIS
- run: |
    SHORT_SHA=$(git rev-parse --short HEAD)
    BRANCH="${GITHUB_REF_NAME}"
    if [[ "${{ github.event_name }}" == "release" ]]; then
      TAG="${{ github.event.release.tag_name }}"
      ...
    fi
    echo "tags=$REG/$IMG:$TAG,$REG/$IMG:$SHORT_SHA" >> $GITHUB_OUTPUT
```

`docker/metadata-action@v6` replaces that whole block with eight lines of YAML and gives you OCI image labels for free.

```yaml
- name: Generate image metadata
  id: meta
  uses: docker/metadata-action@v6
  with:
    images: |
      ${{ steps.ecr-login.outputs.registry }}/${{ env.IMAGE_NAME }}
      ghcr.io/${{ github.repository }}                       # Multi-registry: same tags pushed twice
    tags: |
      type=sha,prefix=,format=long                            # 40-char immutable digest-of-source
      type=sha,format=short                                   # sha-abc1234 (12 chars), human-friendly
      type=ref,event=branch                                   # main, feature-foo
      type=ref,event=pr                                       # pr-123
      type=semver,pattern={{version}}                         # v1.2.3 -> 1.2.3
      type=semver,pattern={{major}}.{{minor}}                 # v1.2.3 -> 1.2
      type=semver,pattern={{major}}                           # v1.2.3 -> 1
      type=raw,value=latest,enable={{is_default_branch}}      # only on main
    labels: |
      org.opencontainers.image.title=My App
      org.opencontainers.image.vendor=ACME Inc.
```

### What the action emits

The action writes two outputs:

```yaml
steps.meta.outputs.tags
# myreg/img:1.2.3
# myreg/img:1.2
# myreg/img:1
# myreg/img:latest
# myreg/img:main
# myreg/img:abc1234567890def...
# myreg/img:sha-abc1234

steps.meta.outputs.labels
# org.opencontainers.image.created=2026-05-07T14:23:11.453Z
# org.opencontainers.image.revision=abc1234567890def...
# org.opencontainers.image.source=https://github.com/my-org/my-repo
# org.opencontainers.image.version=1.2.3
# org.opencontainers.image.title=My App
# org.opencontainers.image.vendor=ACME Inc.
```

You feed both straight into `docker/build-push-action`'s `tags:` and `labels:` inputs (newline-separated lists are accepted natively).

### The semver trio -- the canonical pattern

```
type=semver,pattern={{version}}            # 1.2.3
type=semver,pattern={{major}}.{{minor}}    # 1.2
type=semver,pattern={{major}}              # 1
```

A single push of `git tag v1.2.3` produces three image tags. Why three? **Different consumers want different stability guarantees.** A nervous consumer pins to `myimg:1.2.3` (gets exactly the bytes you released, never silently changes). A moderate consumer pins to `myimg:1.2` (gets patch upgrades like 1.2.4 automatically, no minor surprises). An adventurous consumer pins to `myimg:1` (gets minor upgrades too). Hand them all out and let each consumer pick. This is the exact pattern Docker Hub itself uses for official images (`postgres:16`, `postgres:16.2`, `postgres:16.2.3`).

### The `latest` tag -- conditional on default branch

```
type=raw,value=latest,enable={{is_default_branch}}
```

`is_default_branch` evaluates to `true` only when the workflow is running on `main` (or whatever the repo's default branch is). This is what prevents `latest` from being repointed to a feature branch's image -- a classic foot-gun where you push from a `feat/foo` branch with `--tag latest` and silently break every consumer using `:latest` in production.

### OCI labels -- why `org.opencontainers.image.source` matters

```
org.opencontainers.image.source=https://github.com/my-org/my-repo
```

This is the label that makes the "View source" / "Code" link work on GHCR's package page (and on Docker Hub if you've enabled it). It's also what tools like Renovate, Dependabot, and Trivy use to look up changelogs and CVE metadata. Skipping it isn't a security issue -- it's a "your image looks like a stranger on the registry" issue. Always include it; `metadata-action` adds it automatically when it knows the source.

### Tag-strategy decision framework

| Tag style | When to use | Mutability | Production-safe? |
|---|---|---|---|
| `sha256:abc...` (digest) | K8s manifests, Cosign signing, ArgoCD | Immutable (cryptographic) | YES -- the gold standard |
| `sha-abc1234` (short SHA) | Build-artifact identifiers, dashboards | Immutable (ref-bound) | YES -- but use digest for runtime |
| `1.2.3` (full semver) | Stable consumer pinning | Immutable by convention | YES |
| `1.2` (minor semver) | Auto-patch consumers | Mutable (re-pointed on patch release) | YES, with intent |
| `1` (major semver) | Adventurous auto-update consumers | Mutable (re-pointed on minor release) | YES, with intent |
| `main`, `feature-foo` (branch) | Dev environments, review apps | Mutable | NO for prod |
| `pr-123` (PR ref) | Ephemeral PR review apps | Mutable, short-lived | NO -- PR images are throwaway |
| `latest` | Hobby / demo only | Mutable, no traceability | NO for prod, ever |

**Rule of thumb: production manifests reference digests. CI builds reference semver tags. Dashboards reference short SHAs. `latest` is for tutorials.**

---

## Part 3: BuildKit Cache -- The Larder, Scoped Per Recipe

A cold container build downloads the base image, re-runs every `RUN apt-get install`, re-runs `npm ci` or `pip install` from scratch. On a typical Node app this is 4-8 minutes of wasted runner time per build. BuildKit's layer cache cuts that to 30-90 seconds when warm. The mechanism is the `cache-from` / `cache-to` inputs on `docker/build-push-action@v7`.

### Three cache backends -- the decision

| Backend | Where stored | Size limit | Cross-fork? | Best for |
|---|---|---|---|---|
| `type=gha` | GitHub Actions cache (10 GB / repo, shared with `actions/cache`) | 10 GB total | No (branch-scoped) | Default; small to medium images |
| `type=registry` | A separate tag in your registry (e.g. `myrepo:buildcache`) | Unlimited (you pay for storage) | Yes (anyone with pull) | Monorepo with images >2 GB; cross-fork PR caches |
| `type=inline` | Embedded in the image manifest itself | Bloats the image | Yes (free) | Simplest; only `mode=min`; image consumers re-download cache layers |

**Default to `type=gha`** for small-to-medium images. **Switch to `type=registry`** when (a) image >2 GB, (b) monorepo where the 10 GB GHA cache can't fit all images simultaneously, or (c) you need PRs from forks to share cache with main (forks can read but can't write the GHA cache because of GitHub's branch-scoping).

### `mode=max` vs `mode=min` -- save everything or just the final layers?

```yaml
cache-to: type=gha,mode=max,scope=api    # Cache all stages, including intermediate builders
# vs
cache-to: type=gha,mode=min,scope=api    # Cache only the final image's layers
```

In a multi-stage Dockerfile (the canonical Node pattern: `node:20-alpine` builder stage that runs `npm ci && npm run build`, then a `node:20-alpine` runner stage that copies `dist/`), `mode=min` only caches the final `runner` stage's layers. The next build re-runs the `builder` stage from scratch -- which is exactly the heavy stage with `npm ci`. **`mode=max` caches the builder stage too**, so a re-build with the same `package-lock.json` skips `npm ci` entirely. The cost is cache size: typically 2-3x bigger.

For monorepos approaching the 10 GB GHA cache limit, `mode=min` is a tactical compromise. Otherwise default to `mode=max`.

### `scope=` -- the most-overlooked tuning dial

```yaml
# WRONG -- four images sharing one cache scope, evicting each other
- name: Build api
  uses: docker/build-push-action@v7
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
- name: Build worker
  uses: docker/build-push-action@v7
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max         # OVERWRITES the api cache
- name: Build migrator
  uses: docker/build-push-action@v7
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max         # OVERWRITES the worker cache
```

Without `scope=`, BuildKit defaults to a single scope name `buildkit` per repo. **Every image's cache evicts the previous image's cache.** On a monorepo with 4 images, you cache-hit zero of them on the next run. This is the canonical "we have caching enabled but our pipeline still takes 10 minutes" bug.

```yaml
# RIGHT
- name: Build api
  uses: docker/build-push-action@v7
  with:
    cache-from: type=gha,scope=api
    cache-to: type=gha,mode=max,scope=api
- name: Build worker
  uses: docker/build-push-action@v7
  with:
    cache-from: type=gha,scope=worker
    cache-to: type=gha,mode=max,scope=worker
```

Now each image has its own cache compartment in the larder. The 10 GB pool partitions across them, and they don't fight.

### Multiple cache sources -- read from many, write to one

```yaml
cache-from: |
  type=gha,scope=api
  type=registry,ref=ghcr.io/my-org/my-app:buildcache-api
cache-to: type=gha,mode=max,scope=api
```

You can read from multiple cache sources -- BuildKit tries each in order and uses the most-recent matching layer. You typically write to only one (writing to two backends doubles your cache-publish time and rarely helps).

A common production pattern: **read from both GHA and registry cache, write only to GHA on PR builds, write to both on main-branch builds.** This means main-branch builds populate the registry cache (which forks can then read), and PR builds get cheap reads from GHA without polluting the long-lived registry cache.

### The immutability rule

**Once a cache key (or BuildKit cache layer) is written, it cannot be overwritten -- only read or evicted.** BuildKit's GHA cache backend writes content-addressed layers, so "the same source produces the same layer key" is enforced; you don't normally hit immutability friction the way you do with `actions/cache`. But the consequence is identical: to *refresh* the cache, you bump something that goes into the key. With BuildKit that's the layer's content (changing `package-lock.json` invalidates the `npm ci` layer); with `actions/cache` you bump a `-v2` segment.

### Cache eviction

- **GHA cache:** 10 GB per repo, LRU eviction, **7 days of inactivity** evicts a cache regardless of size.
- **Registry cache:** Lives forever unless you set a registry retention policy. ECR's lifecycle policies can clean up old buildcache tags (`tag prefix: buildcache-` + `count more than 5` is a typical rule).

### Cache backend decision -- the canonical table

| Question | `type=gha` | `type=registry` | `type=inline` |
|---|---|---|---|
| Setup complexity | Trivial (auto-wired) | Medium (extra ref) | Trivial |
| Size limit | 10 GB / repo | Unlimited (you pay) | Bloats image |
| `mode=max` supported | Yes | Yes | NO -- min only |
| Cross-fork share | NO | YES | YES |
| Persistence | 7 days idle | Until you delete | As long as image exists |
| Best for | Default | Monorepo, forks need cache | Simple single-image repos |

**Default: `type=gha,mode=max,scope=<image-name>`. Switch to `type=registry` when you outgrow it.**

---

## Part 4: OIDC in AWS -- The Security Spine

This is the section that, if you internalize nothing else from today's doc, will pay back the most over your career. **OIDC federation is how every modern team authenticates GitHub Actions to AWS.** Long-lived `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` in GitHub secrets is an anti-pattern -- it fails IAM Access Analyzer best-practices checks, leaks in workflow logs if you forget `::add-mask::`, has terrible rotation hygiene (manual, easy to forget), and grants the same blast radius to every workflow that consumes it. OIDC fixes all four.

### The four moving parts

```
+------------------------------+        +-------------------------------+
| GitHub Actions runner        |        | AWS account                   |
|                              |        |                               |
| 1. permissions: id-token: w  |        | 2. IAM identity provider      |
|    -> mint OIDC JWT          |        |    URL: token.actions.        |
|                              |        |         githubusercontent.com |
|              v               |        |    Audience: sts.amazonaws    |
|              v               |        |              .com             |
|    +-----------------+       |        |              ^                |
|    | configure-aws-  |       | JWT    |              |                |
|    | credentials@v6  +-------+--------+--+           |                |
|    +-----------------+       |        |  |  3. IAM role with trust    |
|              v               |        |  |     policy:                |
|       AWS_ACCESS_KEY_ID etc. |<-------+--+   - Federated principal    |
|       (15-60 min lifetime)   |        | STS   = the identity provider |
|                              |        | creds - sub claim must match  |
|                              |        |       4. Role permissions     |
|                              |        |          (ECR push, S3, etc.) |
+------------------------------+        +-------------------------------+
```

1. **Mint the OIDC token in GitHub.** `permissions: id-token: write` at the workflow or job level tells the runner to mint a JWT signed by GitHub's OIDC issuer. Without this line, no token is minted, and `aws-actions/configure-aws-credentials` fails with `Could not load credentials from any providers`. This is THE #1 OIDC foot-gun -- and the error message is unhelpful enough that engineers spend 30 minutes blaming IAM before realizing they forgot the `permissions:` block.

2. **One-time-per-account: create the IAM identity provider.** Under IAM > Identity providers, add a new OpenID Connect provider with URL `https://token.actions.githubusercontent.com` and audience `sts.amazonaws.com`. Provision it once per AWS account; any number of repos and roles can reference it.

3. **Per-role: write the trust policy.** This is the security gate. Whatever you put in the `sub` claim condition decides who can assume the role.

4. **Per-role: write the permission policy.** Standard IAM (e.g. `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:PutImage`, etc. for ECR push). Independent of OIDC.

### The IAM identity provider -- the actual JSON / Terraform

```hcl
# Terraform (preferred)
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["ffffffffffffffffffffffffffffffffffffffff"]   # AWS computes; pinning is optional in modern AWS but allowed for defense-in-depth
}
```

Note: as of 2023, AWS no longer requires the `thumbprint_list` to match GitHub's actual cert thumbprint -- AWS handles validation server-side using IAM's known list of GitHub's certs. You can pass `["ffff..."]` (40 f's) as a placeholder; many Terraform modules still enforce a specific value, which causes occasional "stale thumbprint" issues. Modern Terraform AWS provider docs note this is no longer load-bearing.

### The IAM trust policy -- the four `sub` formats

This is the most error-prone part of OIDC. Memorize these four formats and which is dangerous.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

**The four `sub` formats and what they mean:**

| `sub` value | What it means | Production verdict |
|---|---|---|
| `repo:org/repo:ref:refs/heads/main` | Workflow running on a push or workflow-dispatch on `main` | SAFE -- the production default for branch-based deploys |
| `repo:org/repo:environment:production` | Workflow running with `environment: production` declared | SAFER -- combines with GitHub Environment protection rules (required reviewers, branch restrictions, wait timers) |
| `repo:org/repo:ref:refs/tags/v*` (StringLike with wildcard) | Workflow on a tag matching pattern | SAFE -- the canonical pattern for release-on-tag pipelines |
| `repo:org/repo:pull_request` | Workflow triggered by a pull request | DANGEROUS -- this is the SAME `sub` for every PR, INCLUDING from forks. A role scoped to this `sub` is effectively assumable by any contributor's PR, regardless of branch or author. The classic OIDC privilege-escalation foot-gun. |
| `repo:org/repo:*` | Wildcard for any workflow on any ref | DOUBLE DANGEROUS -- same as `pull_request` plus any branch, any tag, any dispatch. This is "I have given my AWS account to anyone who can open a PR." |

### `StringEquals` vs `StringLike` -- the wildcard discipline

```json
"StringEquals": {
  "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:ref:refs/heads/main"
}
```

For an exact-match `sub`, use `StringEquals`. For wildcards (e.g., any tag matching `v*`), use `StringLike`:

```json
"StringLike": {
  "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:ref:refs/tags/v*"
}
```

**Never use `StringLike` with wildcards on `repo:` itself.** `repo:my-org/*` would let any of your org's repos assume the role -- defeating per-repo isolation.

### 2026 update: immutable subject claims (April 23, 2026)

> **Heads-up for any trust policy you write today.** GitHub announced immutable subject claims for OIDC tokens on **April 23, 2026** -- a security improvement that defends against repo-rename and org-rename attacks. The new `sub` format appends opaque numeric IDs after the org and repo names: `repo:octocat-123456/my-repo-456789:ref:refs/heads/main` (where `123456` is the org ID and `456789` is the repo ID -- both immutable for the lifetime of the entity).
>
> Why it matters: previously, if you renamed `my-org` to `evil-org` (or transferred a repo), an attacker who registered a new `my-org` could mint tokens with the same `sub` your trust policy already accepts. The numeric IDs eliminate that re-registration attack.
>
> **Timeline**: existing repos must opt in via repository settings; **all repositories created or transferred after June 18, 2026 use the new format by default**. Plan accordingly:
>
> 1. **For new repos created after June 18, 2026**: write trust policies in the new format from day one (`repo:my-org-123456/my-repo-789012:...`).
> 2. **For existing repos**: write a forward-compatible trust policy that accepts both formats during the transition. Use a `StringLike` condition with a wildcard segment, or two separate `StringEquals` entries in a list. The brittle path is `StringEquals` on the legacy format only -- the day you opt in, your trust policy silently breaks and every workflow run starts failing with `Not authorized to perform sts:AssumeRoleWithWebIdentity`.
> 3. **For shared org-wide roles**: pull the org and repo numeric IDs from the GitHub API once and bake them into your IAM module's variables, so the trust policy is generated automatically from the IDs rather than the names.
>
> Reference: [GitHub Changelog -- Immutable subject claims for GitHub Actions OIDC tokens](https://github.blog/changelog/2026-04-23-immutable-subject-claims-for-github-actions-oidc-tokens/).

### `permissions:` -- the four lines that matter for container CI

The single biggest security lever you control on a workflow file. Default `GITHUB_TOKEN` permissions vary by org/repo setting (some orgs default to `read-all`, some to `write-all`); **always declare an explicit `permissions:` block** at the **job level** (not workflow level) to minimize blast radius if the job's code is ever compromised.

| Permission | Why this job needs it | Footgun if missing | Footgun if over-granted |
|---|---|---|---|
| `contents: read` | Default for `actions/checkout` to clone the repo | Checkout fails | Mostly benign at `read`; at `write` the workflow can push to the repo |
| `id-token: write` | Mints the OIDC JWT used for AWS OIDC AND for Cosign keyless signing | `Could not load credentials from any providers` (AWS) or `unable to obtain id-token` (Cosign) | Token is workflow-scoped and short-lived; over-grant blast radius is bounded but the token can still be exfiltrated by malicious PR code (see attack chain below) |
| `packages: write` | Required to push images to GHCR (NOT needed for ECR) | `denied: write_package` 403 from GHCR | Workflow can publish or delete packages org-wide |
| `security-events: write` | Required by `github/codeql-action/upload-sarif@v4` to write findings into Code Scanning | `upload-sarif` step silently completes but nothing appears in the Security tab | Workflow can write/dismiss Code Scanning alerts |
| `attestations: write` | Required by `actions/attest-build-provenance@v4` to write to the GitHub Attestations API | Attestation publish fails | Workflow can publish attestations org-wide |

The minimal set for a typical container-CI job in this doc:

```yaml
permissions:
  contents: read         # checkout
  id-token: write        # OIDC for AWS + Cosign
  packages: write        # only if pushing to GHCR; OMIT for ECR-only
  security-events: write # SARIF upload
  attestations: write    # only if using attest-build-provenance
```

**Always declare at job level, not workflow level.** A workflow-level grant leaks into every job, including jobs that don't need it (a typical workflow has a `lint` job, a `build` job, and a `deploy` job -- only `build` needs `packages: write`, only `deploy` needs `id-token: write`).

### The `pull_request` foot-gun, expanded

The trap is subtle enough to deserve its own treatment. The `sub` claim for a PR-triggered workflow is **literally the string `repo:org/repo:pull_request`** -- it does NOT include the PR number, branch, or author. Every PR (from a maintainer or from a fork) produces the same `sub`. So a trust policy with `"sub": "repo:my-org/my-repo:pull_request"` is equivalent to "any contributor who can open a PR can assume this role." Combined with default GitHub `pull_request` triggers running on fork branches, this means **a stranger forking your repo and opening a PR can run any workflow code they wrote and assume your AWS role.**

The fixes:

1. **Don't grant production roles to PR triggers, ever.** PR workflows should at most assume a sandbox role with no write access to anything that matters.
2. **Use environment protection.** `repo:org/repo:environment:staging` requires the workflow to declare `environment: staging`, which then triggers GitHub's environment protection rules (manual approver, branch restriction, wait timer). A PR from a fork can't approve itself.
3. **Use `pull_request_target` only when you know what you're doing.** It runs in the *base* repo's context with full secret access -- and is the source of half of all GitHub Actions supply-chain incidents. Covered in detail in the May 5 advanced-workflows doc; the **OIDC-token-exfiltration attack chain** that connects `pull_request_target` to today's topic is expanded in the next subsection.

### Worked attack chain: how a malicious PR exfiltrates an OIDC token

The full attack that connects yesterday's `pull_request_target` foot-gun to today's OIDC topic. Suppose your trust policy allows `repo:my-org/my-repo:pull_request` (gotcha #3) and you have a workflow on `pull_request_target` that checks out the PR head:

1. Attacker forks `my-repo` and opens a PR with malicious build code.
2. `pull_request_target` triggers the workflow in the **base repo's context** -- it has `id-token: write` and access to all your secrets.
3. The workflow checks out the PR head SHA (the attacker's code) via `actions/checkout@v4` with `ref: ${{ github.event.pull_request.head.sha }}`.
4. The attacker's build script reads two env vars that GitHub injects automatically when `id-token: write` is granted: `ACTIONS_ID_TOKEN_REQUEST_URL` and `ACTIONS_ID_TOKEN_REQUEST_TOKEN`.
5. The script `curl`s `${ACTIONS_ID_TOKEN_REQUEST_URL}&audience=sts.amazonaws.com` with the bearer token to mint a valid JWT signed by GitHub for `sub: repo:my-org/my-repo:pull_request`.
6. The attacker exfiltrates the JWT to their own server.
7. From their server, the attacker calls `sts:AssumeRoleWithWebIdentity` against your AWS account using the stolen JWT. Because your trust policy accepts `pull_request` `sub`, AWS hands them temporary credentials for your role.
8. Attacker now has your AWS keys -- valid for the role's `MaxSessionDuration` (default 1 hour). They use them to dump S3, exfiltrate ECR images, mint IAM users, etc.

The two independent failures that have to compound:

- **Trust policy too permissive** (today's gotcha): accepts `pull_request` `sub`.
- **Workflow uses `pull_request_target` + checks out PR code** (May 5's gotcha): runs untrusted code with secrets.

Either fix alone breaks the chain. Belt-and-suspenders defense: scope `sub` to `ref:refs/heads/main` or `environment:production`, AND never check out PR code in `pull_request_target`. This is why the May 5 doc and today's doc are paired -- the security model is one layered system, not two.

### `aws-actions/configure-aws-credentials@v6`

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v6
  with:
    role-to-assume: arn:aws:iam::123456789012:role/GHA-MyRepo-DeployRole
    role-session-name: GitHubActions-${{ github.run_id }}-${{ github.run_attempt }}
    aws-region: us-east-1
    role-duration-seconds: 1800                # 30 min; default is 60 min
    role-skip-session-tagging: false           # Default false; tags are useful
    # role-chaining: true                      # When chaining from one role to another
```

What it does under the hood:

1. Reads the OIDC JWT from the runner's `ACTIONS_ID_TOKEN_REQUEST_URL` and `ACTIONS_ID_TOKEN_REQUEST_TOKEN` env vars (set automatically when `id-token: write` is granted).
2. Calls `sts:AssumeRoleWithWebIdentity` with the JWT, the role ARN, and the session name.
3. AWS validates the JWT against the trust policy's `sub` and `aud` conditions.
4. STS returns temporary credentials (1-hour default lifetime).
5. The action exports `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`, `AWS_REGION`, and `AWS_DEFAULT_REGION` as workflow env vars (via `core.exportVariable`), so all subsequent steps in the job see them.

### Verify OIDC actually worked

```yaml
- name: Verify AWS identity
  run: aws sts get-caller-identity
```

The output should show:

```
{
    "UserId": "AROAEXAMPLE:GitHubActions-7894561230-1",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/GHA-MyRepo-DeployRole/GitHubActions-7894561230-1"
}
```

The `assumed-role` ARN tells you which role was assumed AND the session name. **If you don't see this output, OIDC did not work; debugging starts at `permissions: id-token: write` then moves to the trust policy `sub` claim.**

### `role-session-name` -- CloudTrail attribution

The `role-session-name` field is what shows up in CloudTrail under `userIdentity.arn`. **Always include `${{ github.run_id }}`** so you can pivot from a CloudTrail event back to the exact workflow run that caused it. A common production pattern:

```yaml
role-session-name: GHA-${{ github.repository }}-${{ github.run_id }}-${{ github.run_attempt }}
```

(AWS limits session names to 64 chars and `[a-zA-Z0-9._=,@-]+`; `github.repository` includes a `/` which is invalid -- replace it or just use `github.run_id`.)

### Session tagging -- defense in depth

```yaml
- uses: aws-actions/configure-aws-credentials@v6
  with:
    role-to-assume: arn:aws:iam::123456789012:role/GHA-MyRepo-DeployRole
    role-skip-session-tagging: false        # Default; explicit for clarity
    aws-region: us-east-1
```

When session tagging is on (default), the action attaches the JWT claims as STS session tags: `aws:RequestTag/repository=my-org/my-repo`, `aws:RequestTag/workflow=release.yml`, `aws:RequestTag/actor=alice`, etc. **You can then write IAM policies that condition on these tags:**

```json
{
  "Effect": "Allow",
  "Action": "ecr:PutImage",
  "Resource": "arn:aws:ecr:us-east-1:123456789012:repository/api",
  "Condition": {
    "StringEquals": {
      "aws:PrincipalTag/repository": "my-org/my-repo"
    }
  }
}
```

This is defense-in-depth: even if someone misconfigures the trust policy and a wider set of workflows can assume the role, the *permission* policy further restricts what each session can do based on its tags. This pattern is invaluable for shared roles used by many repos in an org.

> **The two-question framing for trust-policy audits.** When you read a trust policy, separate two concerns: **trust policy answers WHO** (which workflows can assume this role); **identity policy answers WHAT** (which AWS actions a session is allowed to perform). A correctly-scoped trust policy paired with `AdministratorAccess` is still a critical finding -- you've reduced the blast-radius universe to "people who can push to one branch of one repo," but every one of them still gets full admin. Both halves have to be tight. A common failure mode in inherited AWS accounts is a perfectly-scoped trust policy attached to a role that has `*:*` on `Resource: *` -- and the auditor reading only the trust policy declares it safe.

> **The trust policy is doing exactly what it was told.** The other framing that pays off in incident response: when you find a JWT was accepted by AWS that "shouldn't have been," resist the urge to say "AWS validated wrong" or "the trust policy is broken." It almost never is -- AWS only validates `aud` and `sub` against what the policy specified, and the policy specified what it specified. The bug is *upstream*: in the GitHub-side trigger and checkout combination that let an attacker stand in a privileged context and ask GitHub to mint a JWT whose claims happen to match. Other JWT claims (`event_name`, `ref`, `workflow_ref`, `job_workflow_ref`, `environment`) *could* have distinguished the two but weren't part of any trust-policy condition. The fix is layered: tighten the trust policy condition (add `event_name` or `job_workflow_ref`), AND tighten the workflow side (don't combine `pull_request_target` with fork-code checkout, declare `id-token: write` at the *job* level only on jobs that need it).

### Role chaining -- one workflow, two AWS accounts

Some workflows need to assume multiple roles -- e.g., assume a role in the dev account to read a config, then assume a role in the prod account to deploy. Call the action twice with `role-chaining: true` on the second call:

```yaml
- name: Assume dev role
  uses: aws-actions/configure-aws-credentials@v6
  with:
    role-to-assume: arn:aws:iam::111111111111:role/DevReadConfig
    aws-region: us-east-1

- name: Read dev config
  run: aws ssm get-parameter --name /shared/config

- name: Assume prod role (chained)
  uses: aws-actions/configure-aws-credentials@v6
  with:
    role-to-assume: arn:aws:iam::222222222222:role/ProdDeploy
    aws-region: us-east-1
    role-chaining: true                # Reuse current creds instead of OIDC token

- name: Deploy
  run: aws ecs update-service ...
```

`role-chaining: true` makes the second call use the existing AWS credentials (from the dev role) to call `sts:AssumeRole` on the prod role, instead of using the OIDC JWT again. The prod role's trust policy must trust the dev role (not GitHub's OIDC provider directly). This is the standard pattern when a repo's prod role is in a separate AWS Organizations account.

---

## Part 5: ECR vs GHCR -- The Auth Difference

Two registries dominate GitHub Actions container CI: AWS ECR (when you're deploying to AWS) and GHCR (GitHub's own container registry, when you're building images consumed by other GitHub repos or by Kubernetes clusters). The auth model differs.

### ECR auth -- via OIDC + `amazon-ecr-login`

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v6
  with:
    role-to-assume: arn:aws:iam::123456789012:role/GHA-Deploy
    aws-region: us-east-1

- name: Login to ECR
  id: ecr-login
  uses: aws-actions/amazon-ecr-login@v2
  # No `with:` needed for the basic ECR private case

# Now docker/login-action is NOT needed for ECR -- amazon-ecr-login already
# configured the docker daemon's auth.json. You feed the registry into
# build-push-action's tags directly.
```

`aws-actions/amazon-ecr-login@v2` calls `ecr:GetAuthorizationToken`, gets a 12-hour Docker auth token, and writes it to the runner's docker config. **It also emits a `registry` output** (`123456789012.dkr.ecr.us-east-1.amazonaws.com`) that you can interpolate into image refs.

### GHCR auth -- via the built-in `GITHUB_TOKEN`

```yaml
permissions:
  packages: write                  # Required for GHCR push
  contents: read

# ...

- name: Login to GHCR
  uses: docker/login-action@v4
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
```

GHCR uses the workflow's auto-minted `GITHUB_TOKEN` (which is automatically scoped to the repo). You need `permissions: packages: write` to push; reads from public GHCR don't require any auth.

### The auth comparison

| Feature | ECR | GHCR | Docker Hub |
|---|---|---|---|
| Auth mechanism | OIDC -> STS -> 12h docker token | `GITHUB_TOKEN` (auto-minted) | Long-lived PAT in secrets |
| Long-lived secrets needed? | NO (with OIDC) | NO | YES (PAT) |
| Setup complexity | Medium (IAM trust policy) | Trivial | Trivial |
| Cross-account / cross-org auth | Standard IAM | Limited (org packages) | Pull-through registries |
| Cost | $0.10/GB-month, $0.09/GB egress | Free for public, included in plan for private | Free tier limited; paid for orgs |
| Best for | AWS-native deploys (EKS, ECS, Lambda) | Open-source projects, internal tools used across repos | Public images for community |

In practice: **ECR for prod images deployed to AWS infrastructure; GHCR for everything else.** Many setups push the same image to both -- `metadata-action` makes that one-line ("`images:`" accepts multiple registries).

---

## Part 6: Trivy -- The X-Ray Scanner

Container vulnerability scanning is a non-negotiable production gate. **Trivy is the de-facto open-source scanner**, with native GitHub Actions integration via `aquasecurity/trivy-action@v0.36.0`. It scans for OS package vulnerabilities (Alpine APK CVEs, Debian apt CVEs), application library vulnerabilities (Node `package-lock.json`, Python `requirements.txt`, Go `go.sum`, Java `pom.xml`), leaked secrets in image layers, and Dockerfile misconfigurations -- all in one action.

### Three production patterns

#### Pattern 1: Single hard gate (the minimum)

```yaml
- name: Trivy scan -- block on CRITICAL/HIGH
  uses: aquasecurity/trivy-action@v0.36.0
  with:
    image-ref: ${{ steps.ecr-login.outputs.registry }}/api:${{ steps.meta.outputs.version }}
    severity: 'CRITICAL,HIGH'
    exit-code: '1'                # Stop the workflow on findings
    ignore-unfixed: true          # Don't fail on CVEs with no patch available
    vuln-type: 'os,library'
```

Translated: scan the freshly-built image; if any CRITICAL or HIGH severity vulnerability with an available fix is found, fail the workflow (exit 1). `ignore-unfixed: true` is the moral hygiene flag -- you cannot act on a CVE that has no patch yet, so failing on it just blocks shipping without giving you a path forward.

This is the minimum useful gate. Default for `release.yml` workflows that publish images.

#### Pattern 2: Two-tier gate (the production sweet spot)

```yaml
- name: Trivy scan -- SARIF output (informational, never fails)
  uses: aquasecurity/trivy-action@v0.36.0
  with:
    image-ref: ${{ steps.ecr-login.outputs.registry }}/api:${{ steps.meta.outputs.version }}
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL'
    exit-code: '0'                # Never fail; this is reporting only

- name: Upload SARIF to Code Scanning
  uses: github/codeql-action/upload-sarif@v4    # NOTE: v4, not v3
  if: always()                                   # Upload even if next step fails
  with:
    sarif_file: 'trivy-results.sarif'
    category: 'trivy-image'

- name: Trivy scan -- hard gate on CRITICAL/HIGH
  uses: aquasecurity/trivy-action@v0.36.0
  with:
    image-ref: ${{ steps.ecr-login.outputs.registry }}/api:${{ steps.meta.outputs.version }}
    severity: 'CRITICAL,HIGH'
    exit-code: '1'
    ignore-unfixed: true
```

The two-tier pattern runs Trivy twice. The first run captures *all* findings as SARIF (uploaded to GitHub Code Scanning, where they appear as annotations on PR diffs in the Security tab); the second run is the hard gate that actually fails the build on CRITICAL/HIGH. **Why two passes?** Because you want LOW/MEDIUM findings *visible* (so reviewers see them, dependabot opens upgrade PRs, the vuln team has a queue to triage) without *blocking* (because forcing a developer to fix every LOW CVE in a transitive dep that landed an hour ago is how you get developers turning off the scanner entirely).

**Critical version note**: `github/codeql-action/upload-sarif@v4` is the current major (v4.35.3 as of May 2026). v3 still receives parallel patches but new pipelines should target v4. Many older blog posts pin `@v3`; upgrade them.

#### Pattern 3: Allowlisted CVEs via `.trivyignore`

```
# .trivyignore (at repo root)
# Format: CVE-ID with optional comment

CVE-2024-12345 # accepted: only triggered in non-default config; review by 2026-08-01
CVE-2024-67890 # accepted: vendor patch lands in next release; track in JIRA-1234
CVE-2024-11111 exp:2026-09-30 # auto-expire (not enforced by Trivy itself; honor by review)
```

Each allowlisted CVE should have:
1. **A reason** (why it's being accepted -- not exploitable in this config? Mitigated by infra control? Vendor-fix-pending?)
2. **A review date** (forces re-evaluation; without this, the allowlist becomes a graveyard)
3. **A ticket reference** (so the suppression has an audit trail)

Treat `.trivyignore` like a security policy file: code review every change to it (the same way you'd review changes to `iam-policy.json`).

### Critical Trivy flags

| Flag | What it controls | Default | When to override |
|---|---|---|---|
| `severity` | Severities to report/fail | `UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL` | Hard gate: `CRITICAL,HIGH` only |
| `exit-code` | Exit code on findings | `0` (don't fail) | `1` for hard gate |
| `ignore-unfixed` | Skip CVEs with no patch | `false` | `true` for hard gate (you can't act on them) |
| `vuln-type` | What to scan | `os,library` | Rarely change |
| `scanners` | Which scanners to run | `vuln,secret` | Add `misconfig` for IaC repos |
| `cache-dir` | CVE DB cache location | None (re-downloads each run) | Always set; saves ~600 MB / run |
| `format` | Output format | `table` | `sarif` for Code Scanning, `json` for tools |
| `output` | Output file | None (stdout) | Required when uploading SARIF |
| `timeout` | Per-image scan timeout | `5m` | Increase for large images |

### `cache-dir` -- save the 600 MB CVE DB

Trivy downloads a ~600 MB CVE database on every run by default. With many CI runs per day, this is bandwidth waste and adds 30-90 seconds per scan. Cache it:

```yaml
- name: Cache Trivy DB
  uses: actions/cache@v4
  with:
    path: ~/.cache/trivy
    key: trivy-db-${{ runner.os }}-${{ hashFiles('.github/trivy-cache-version') }}
    restore-keys: |
      trivy-db-${{ runner.os }}-

- name: Trivy scan
  uses: aquasecurity/trivy-action@v0.36.0
  with:
    image-ref: ...
    cache-dir: ~/.cache/trivy
    severity: 'CRITICAL,HIGH'
    exit-code: '1'
```

The `.github/trivy-cache-version` file is a manual cache-buster; bump its contents (e.g., to `v2`) when you want to force a fresh DB pull.

### Trivy gating decision framework

| Trigger | Severity | exit-code | SARIF upload | Why |
|---|---|---|---|---|
| `release.yml` (push to main, tag) | CRITICAL,HIGH | 1 | Yes | Production images must be clean of actionable CVEs |
| `pr.yml` on application code PRs | CRITICAL,HIGH | 1 (or 0 for transitive grace period) | Yes | Block merging code that introduces vulns |
| `pr.yml` on documentation-only PRs | n/a | n/a | n/a | Skip the scan via path filter |
| `nightly.yml` | LOW,MEDIUM,HIGH,CRITICAL | 0 | Yes | Catch newly-disclosed CVEs in already-built images; informational only |

The nightly pattern is underrated: a CVE disclosed today against a base image you built and pushed yesterday WILL NOT trigger your release workflow (no code changed). A nightly Trivy scan against the *latest pushed image* surfaces this drift. Pair it with auto-rebuild-and-push on findings.

---

## Part 7: BuildKit Attestations -- The Receipt That Travels With the Loaf

For SLSA Level 3 / SOC 2 / FedRAMP / supply-chain compliance, you need provable answers to "what's inside this image?" (SBOM) and "how was this image built?" (provenance). BuildKit can attach both as **in-toto attestations bundled into the OCI image manifest** -- one image artifact, one digest, with the SBOM and SLSA provenance attached as separate manifests under a top-level OCI image index.

### The `provenance:` and `sbom:` flags

```yaml
- name: Build and push
  uses: docker/build-push-action@v7
  with:
    context: .
    push: true
    tags: ${{ steps.meta.outputs.tags }}
    labels: ${{ steps.meta.outputs.labels }}
    provenance: true                 # SLSA v0.2 provenance attestation
    sbom: true                       # SPDX JSON SBOM, Syft-generated
    # Or unified syntax:
    # attests: |
    #   type=sbom,mode=max
    #   type=provenance,mode=max
```

What this produces:

```
ECR or GHCR
+--------------------------------------------------------+
| OCI image index (manifest list)                        |
|   sha256:abc123...                                     |
|   +- linux/amd64 manifest sha256:def456...             |
|   |    +- config blob                                  |
|   |    +- layer blobs                                  |
|   +- in-toto SBOM attestation manifest sha256:ghi789...|
|   |    media type: application/vnd.in-toto+json        |
|   |    predicate: SPDX 2.3 JSON                        |
|   +- in-toto provenance manifest sha256:jkl012...      |
|        media type: application/vnd.in-toto+json        |
|        predicate: SLSA v0.2 provenance                 |
+--------------------------------------------------------+
```

The whole thing is one image with one root digest (`sha256:abc123...`). Pulling the image pulls only the platform manifest by default; pulling the SBOM/provenance is opt-in.

### Verifying the SBOM

```bash
docker buildx imagetools inspect \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/api:1.2.3 \
  --format '{{ json .SBOM }}'
```

Output is the SPDX 2.3 JSON document listing every package in the image with name, version, license, source URL, and SHA256. Pipe through `jq` to query, e.g., "find all GPL-licensed packages":

```bash
docker buildx imagetools inspect ... --format '{{ json .SBOM }}' \
  | jq '.SPDX.packages[] | select(.licenseConcluded | test("GPL"))'
```

### Verifying the provenance

```bash
docker buildx imagetools inspect \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/api:1.2.3 \
  --format '{{ json .Provenance }}'
```

Output includes the build invocation: which builder version, what Dockerfile, which git commit, which workflow, etc. This is the SLSA Level 3 evidence for "this image was built by THIS pipeline at THIS commit."

### Critical caveats

1. **Registry-only.** Attestations only attach when **pushing to a registry**. `docker buildx build --load` (which loads into the local docker daemon) **strips them** because the local daemon doesn't understand the OCI image index v1.1 format. So you cannot `--load` an attested image; you must `--push`.
2. **Registry support.** ECR Private and GHCR support multi-manifest layouts natively. ECR Public did until mid-2024. Some older registries (Artifactory <7.59, very old Harbor) reject the layout. If your registry rejects with `unsupported media type`, that's the symptom.
3. **Default vs `mode=max`.** `provenance: true` defaults to `mode=min` (only build invocation metadata). `provenance: mode=max` includes the full build dependency graph. `mode=max` slightly inflates the manifest; recommended for production.
4. **`actions/attest-build-provenance@v4` is the GitHub-native alternative.** It writes provenance to GitHub's own attestation API (instead of attaching to the image manifest). Useful when you want provenance separately queryable via the GitHub API. The two are complementary, not exclusive -- many high-security orgs run both.
5. **Builder-driver-first debug discipline.** When `imagetools inspect` returns `null` for `.SBOM` or `.Provenance` despite `provenance: true` and `sbom: true` being set, **suspect the active Buildx builder driver before suspecting the registry, action version, or YAML**. The default `docker` driver (used when `docker/setup-buildx-action` is missing from the workflow) is BuildKit running inside the local Docker daemon -- it writes flat single-manifest images and **does not produce the OCI Image Index v1.1 layout that attestations need to live under**. BuildKit silently drops the requested attestations because there's no Image Index for sibling attestation manifests to attach to. The build succeeds, the action exits 0 with no warnings, the image gets pushed -- and `.SBOM` and `.Provenance` come back `null`. The fix is one missing step: add `docker/setup-buildx-action@v4` before the build to boot a `docker-container`-driver instance and register it as the active builder. Green CI is not evidence that attestations exist; only `imagetools inspect` is.

### When to use BuildKit attestations vs `cosign attest`

| Goal | Use |
|---|---|
| SBOM/provenance attached to the image, accessible via standard OCI tooling | BuildKit `provenance: true` + `sbom: true` |
| Signed SBOM that's independently verifiable on Rekor | `cosign attest --predicate sbom.json --type spdx` (separate step) |
| GitHub-native attestation discovery | `actions/attest-build-provenance@v4` |
| FedRAMP / SLSA Level 3 compliance | All three (belt and suspenders) |

Today's `release.yml` uses the BuildKit method (most ergonomic) and Cosign for signing. Adding `cosign attest` for the SBOM is a one-line extension if you need it.

---

## Part 8: Cosign Keyless Signing -- The Tamper-Evident Holographic Seal

This is the headline security feature of modern container CI. **Cosign keyless signing** removes the need to manage GPG keys or HSMs while producing cryptographically-verifiable signatures bound to the workflow identity. The Sigstore project (which Cosign is part of) provides a free public CA (Fulcio) and transparency log (Rekor); your workflow's OIDC token is the only credential needed.

### The flow

```
+-----------------+              +----------------+              +-----------------+
| GitHub Actions  | OIDC token   | Sigstore       | 10-min cert  | Cosign signs    |
| runner          +------------->+ Fulcio CA      +------------->+ image digest    |
|                 |              |                |              | with ephemeral  |
|                 |              | Binds workflow |              | private key     |
|                 |              | identity to    |              |                 |
|                 |              | ephemeral key  |              |                 |
+-----------------+              +----------------+              +--------+--------+
                                                                          |
                                                                          | sig + cert
                                                                          v
                                              +--------------------+
                                              | OCI registry       |
                                              | (ECR / GHCR)       |
                                              | <image>.sig tag    |
                                              +--------------------+
                                                                          |
                                                                          | log entry
                                                                          v
                                              +--------------------+
                                              | Rekor              |
                                              | transparency log   |
                                              | (public, immutable)|
                                              +--------------------+
```

### Step by step

1. **Install Cosign.** `sigstore/cosign-installer@v4` (v4 is the current major; v3 dropped Node 16 support and was bumped to v4).
2. **Mint the OIDC token.** `permissions: id-token: write` (same token used for AWS).
3. **Sign by digest.** `cosign sign --yes <image>@<digest>`. The `--yes` flag (or `COSIGN_YES=true`) skips the interactive Y/N prompt about uploading to the public Rekor log.
4. **Cosign authenticates to Fulcio** using the GitHub OIDC token. Fulcio mints a short-lived (10-minute) X.509 certificate whose `Subject Alternative Name` is the workflow identity (`https://github.com/my-org/my-repo/.github/workflows/release.yml@refs/heads/main`).
5. **Cosign signs the image digest** using an ephemeral private key generated for this signing operation only.
6. **Cosign pushes the signature** as a `.sig` tag in the same registry alongside the image (e.g., `myreg/img:sha256-abc123.sig` -- the `:` is replaced with `-` to fit OCI tag rules). **The signature path is keyed by the digest, NOT by the reference passed to `cosign sign`.** This is what makes the sign-by-tag attack work: if you signed `myreg/img:v1.2.3` (which resolved to `sha256:ORIGINAL` at signing time), the signature lives at `:sha256-ORIGINAL.sig`. An attacker who later pushes `sha256:EVIL` and re-points `v1.2.3` at it doesn't tamper with your original signature -- it's still sitting in the registry, *orphaned*, because nothing points to `sha256:ORIGINAL` anymore. The verifier resolves `v1.2.3` to its current target (`sha256:EVIL`), looks up the signature at `:sha256-EVIL.sig`, finds whatever the attacker signed there under their own Fulcio identity, and -- without a strict `--certificate-identity-regexp` -- returns success. Sign by digest, always.
7. **Cosign uploads the signing event to Rekor**, the public transparency log. Anyone can independently query Rekor by image digest and verify the signature was minted at a specific time by a specific workflow.
8. **The ephemeral private key is discarded.** No long-lived signing key exists to be stolen.

### The YAML

```yaml
- name: Install Cosign
  uses: sigstore/cosign-installer@v4

- name: Sign image with Cosign
  env:
    COSIGN_YES: 'true'
    IMAGE: ${{ steps.ecr-login.outputs.registry }}/api
    DIGEST: ${{ steps.build.outputs.digest }}
  run: |
    cosign sign \
      --yes \
      "${IMAGE}@${DIGEST}"
```

Note: the image reference is `IMAGE@DIGEST`, NOT `IMAGE:TAG`. **Always sign by digest.** A tag is mutable -- you can re-point `myimg:v1.2.3` to a different image tomorrow -- so signing a tag is signing a sticker that can be peeled off and stuck on a different loaf. The digest is the cryptographic identity of the bytes; sign that.

`docker/build-push-action@v7` exposes the pushed digest as `steps.<id>.outputs.digest`:

```yaml
- name: Build and push
  id: build
  uses: docker/build-push-action@v7
  with: { ... }
# steps.build.outputs.digest = sha256:abc123...
```

### Verifying signatures

```bash
cosign verify \
  --certificate-identity-regexp '^https://github.com/my-org/[^/]+/\.github/workflows/release\.yml@refs/heads/main$' \
  --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' \
  '123456789012.dkr.ecr.us-east-1.amazonaws.com/api@sha256:abc123...'
```

The two flags are the policy gate:

- **`--certificate-identity-regexp`** restricts which workflow identities are accepted. Use the regex anchor `^...$` to prevent prefix matches -- without anchors, `https://github.com/my-org/` matches `https://github.com/my-org-evil-twin/...` too.
- **`--certificate-oidc-issuer`** restricts which OIDC issuer minted the cert. For GitHub Actions it's always `https://token.actions.githubusercontent.com`. For self-hosted GitHub Enterprise Server it's a different URL.

### Admission control -- where verification actually happens

Signing without verification is theater. The point is to enforce verification at admission time -- before pods can pull and run the image. The standard tools:

- **Sigstore Policy Controller** (Kubernetes admission webhook). Configured with a `ClusterImagePolicy` resource that lists allowed identity regexps; rejects pods that pull unsigned or wrong-identity images.
- **Kyverno** with `verifyImages` rules. Same idea, broader feature set.
- **Gatekeeper** + Cosign integration via Rego.

The admission policy typically reads:

```yaml
apiVersion: policy.sigstore.dev/v1alpha1
kind: ClusterImagePolicy
metadata:
  name: my-org-policy
spec:
  images:
    - glob: "123456789012.dkr.ecr.us-east-1.amazonaws.com/**"
  authorities:
    - keyless:
        url: https://fulcio.sigstore.dev
        identities:
          - issuer: https://token.actions.githubusercontent.com
            subjectRegExp: ^https://github.com/my-org/.+/\.github/workflows/release\.yml@refs/heads/main$
```

**This is the pay-off of every step from Part 1 to here.** The image is built, scanned, signed, and pushed by your CI. The cluster admission controller verifies the signature was minted by *this exact workflow* on *the main branch*. An attacker who gets push access to the registry but cannot run your workflow cannot forge a valid signature -- the Fulcio cert is bound to the workflow's OIDC identity, which they don't have. **The seal is what makes "we run only verified images" enforceable.**

### Why keyless > traditional key-based signing

| Aspect | Keyless (Cosign + Fulcio) | Traditional (GPG / KMS) |
|---|---|---|
| Key management | None -- ephemeral keys, discarded | Keys must be stored, rotated, backed up |
| Key compromise | Not possible (ephemeral) | Catastrophic; revocation hard |
| Identity binding | OIDC issuer + SAN URI -- workflow identity | Key fingerprint -- whoever holds the key |
| Audit trail | Rekor transparency log (public, immutable) | Manual logging if any |
| HSM cost | $0 | Thousands of dollars / month |
| Verifiable by anyone | Yes (Rekor + Fulcio root of trust) | Requires distributing public keys |

Keyless removes the entire class of "the signing key was compromised" incidents. The cost is the dependency on Sigstore infrastructure (Fulcio, Rekor) -- which is operated by the OpenSSF and is now backed by Google, Red Hat, GitHub, and others. For air-gapped environments, you can run private Fulcio + Rekor instances.

---

## Part 9: Connecting to GitOps -- Where the Digest Goes Next

Today's pipeline ends at `cosign sign`. **The digest emitted by `docker/build-push-action` is the handoff point** to the GitOps update workflow that will be tomorrow's topic. The full lifecycle:

```
+---------------------+      digest      +---------------------+
| release.yml         +------------------>| update-manifests.yml|
| (this doc)          |  steps.build.    | (or commit hook)    |
| build/scan/sign/push|  outputs.digest  | bumps values.yaml   |
+---------------------+                  | in config repo      |
                                          +----------+----------+
                                                     | PR
                                                     v
                                          +---------------------+
                                          | Config repo PR      |
                                          | merged to main      |
                                          +----------+----------+
                                                     | Argo CD
                                                     v
                                          +---------------------+      cosign verify
                                          | Argo CD reconciles  +---------------------+
                                          | manifests -> K8s    |  (admission webhook)|
                                          +---------------------+ <-------------------+
                                                                  signature OK -> pod runs
                                                                  signature bad -> reject
```

The two patterns for the manifest-update step:

**Pattern A: GitHub Actions opens a PR.** After the build job succeeds, a follow-up job clones the config repo, runs `yq -i '.image.digest = "<DIGEST>"' values.yaml`, opens a PR via `peter-evans/create-pull-request`. PR is reviewed and merged; ArgoCD picks up the change.

**Pattern B: ArgoCD Image Updater watches the registry.** ArgoCD Image Updater (covered in the Apr 17-19 docs) polls the registry for new tags matching configured patterns, writes the new digest to the config repo (or to ArgoCD's internal annotation storage), and triggers a sync. This pattern needs no follow-up CI job -- the registry push itself is the trigger.

Either way, **the digest -- and its signature -- is what flows downstream.** This is why the doc has hammered "sign by digest" throughout: the digest is the load-bearing identifier across the entire CI/CD chain.

---

## Part 10: The Flagship `release.yml` -- End to End

This is the canonical workflow that wires everything from Parts 1-9 into one file. Treat this as the reference; production variants are minor adaptations.

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches: [main]
    tags: ['v*']
  workflow_dispatch:

# Cancel in-flight runs of the same ref to avoid race conditions on tag updates.
concurrency:
  group: release-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: false   # Don't cancel an in-flight push to main mid-build

# Default to read-only at the workflow level; jobs grant more as needed.
permissions:
  contents: read

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: api
  IMAGE_NAME: api

jobs:
  build-scan-sign:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write       # OIDC for AWS AND for Cosign keyless
      packages: write       # If also pushing to GHCR
      security-events: write # SARIF upload to Code Scanning

    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
      image-tag: ${{ steps.meta.outputs.version }}

    steps:
      # ----- 1. Checkout the source -----
      - name: Checkout
        uses: actions/checkout@a1b2c3d4e5f6789012345678901234567890abcd # v4.2.2 SHA-pinned
        with:
          fetch-depth: 0      # Full history for SBOM / provenance accuracy

      # ----- 2. QEMU + Buildx -----
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v4

      # ----- 3. AWS OIDC + ECR login -----
      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GHA-MyRepo-DeployRole
          role-session-name: gha-${{ github.run_id }}-${{ github.run_attempt }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Verify AWS identity
        run: aws sts get-caller-identity

      - name: Login to Amazon ECR
        id: ecr-login
        uses: aws-actions/amazon-ecr-login@v2

      # ----- 4. Generate tags and labels deterministically -----
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v6
        with:
          images: |
            ${{ steps.ecr-login.outputs.registry }}/${{ env.ECR_REPOSITORY }}
          tags: |
            type=sha,prefix=,format=long
            type=sha,format=short
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=semver,pattern={{major}}
            type=raw,value=latest,enable={{is_default_branch}}
          labels: |
            org.opencontainers.image.title=${{ env.IMAGE_NAME }}
            org.opencontainers.image.vendor=ACME Inc.

      # ----- 5. Build + push (with attestations + cache) -----
      - name: Build and push
        id: build
        uses: docker/build-push-action@v7
        with:
          context: .
          file: ./Dockerfile
          push: true
          platforms: linux/amd64
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha,scope=${{ env.IMAGE_NAME }}
          cache-to: type=gha,mode=max,scope=${{ env.IMAGE_NAME }}
          provenance: mode=max
          sbom: true

      # ----- 6. Trivy scan (two-tier gate) -----
      - name: Cache Trivy DB
        uses: actions/cache@v4
        with:
          path: ~/.cache/trivy
          key: trivy-db-${{ runner.os }}-v1
          restore-keys: |
            trivy-db-${{ runner.os }}-

      - name: Trivy SARIF report (informational)
        uses: aquasecurity/trivy-action@v0.36.0
        with:
          image-ref: ${{ steps.ecr-login.outputs.registry }}/${{ env.ECR_REPOSITORY }}@${{ steps.build.outputs.digest }}
          format: sarif
          output: trivy-results.sarif
          severity: 'UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL'
          exit-code: '0'
          ignore-unfixed: false
          cache-dir: ~/.cache/trivy
          scanners: 'vuln,secret,misconfig'

      - name: Upload SARIF to Code Scanning
        if: always()
        uses: github/codeql-action/upload-sarif@v4   # v4, not v3
        with:
          sarif_file: trivy-results.sarif
          category: trivy-image

      - name: Trivy hard gate (CRITICAL/HIGH)
        uses: aquasecurity/trivy-action@v0.36.0
        with:
          image-ref: ${{ steps.ecr-login.outputs.registry }}/${{ env.ECR_REPOSITORY }}@${{ steps.build.outputs.digest }}
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
          ignore-unfixed: true
          cache-dir: ~/.cache/trivy
          scanners: 'vuln,secret'

      # ----- 7. Cosign keyless signing (by DIGEST) -----
      - name: Install Cosign
        uses: sigstore/cosign-installer@v4

      - name: Sign image
        env:
          COSIGN_YES: 'true'
          IMAGE_REF: ${{ steps.ecr-login.outputs.registry }}/${{ env.ECR_REPOSITORY }}@${{ steps.build.outputs.digest }}
        run: |
          cosign sign --yes "${IMAGE_REF}"

      # ----- 8. Job summary for humans -----
      - name: Job summary
        if: always()
        run: |
          {
            echo "## Release Summary"
            echo ""
            echo "| Field | Value |"
            echo "| --- | --- |"
            echo "| Image | ${{ steps.ecr-login.outputs.registry }}/${{ env.ECR_REPOSITORY }} |"
            echo "| Digest | \`${{ steps.build.outputs.digest }}\` |"
            echo "| Tags | ${{ steps.meta.outputs.tags }} |"
            echo "| Workflow run | ${{ github.run_id }} |"
          } >> "$GITHUB_STEP_SUMMARY"

  # Downstream: bump manifest in config repo (placeholder for tomorrow's GitOps doc)
  trigger-deploy:
    needs: build-scan-sign
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "Will bump values.yaml to digest ${{ needs.build-scan-sign.outputs.image-digest }}"
```

### What this workflow guarantees

After this workflow runs successfully, you have:

1. **An image at a deterministic digest** in ECR, tagged with semver + SHA + (conditionally) `latest`.
2. **An SBOM and SLSA provenance attestation** attached to the image's OCI manifest, retrievable via `docker buildx imagetools inspect`.
3. **A SARIF vulnerability report** uploaded to GitHub Code Scanning (visible in the Security tab and as PR diff annotations on subsequent PRs).
4. **No CVEs of CRITICAL or HIGH severity (with available fixes)** in the image (or the workflow failed).
5. **A Cosign signature** in the registry, bound to the workflow identity, with a Rekor transparency-log entry.
6. **Zero long-lived AWS credentials** anywhere in the workflow, repo secrets, or runner state.
7. **The image digest as a workflow output**, ready to be consumed by the next deploy step.

Total runtime: typically 3-7 minutes for a warm-cache build of a 200-500 MB image. Cold-cache first run: 8-15 minutes.

---

## Part 11: The 15 Production Gotchas

These are the lessons that you will find by hitting them in production -- or by reading this list now and avoiding them. Each is a one-line problem with a one-line fix.

1. **`permissions: id-token: write` missing -> `Could not load credentials from any providers`.** OIDC token isn't minted. Add the permissions block at workflow OR job level.

2. **Trust policy `sub` claim wildcard `repo:org/repo:*` -> any workflow on any ref can assume the role.** This includes PR workflows from forks. Always scope to specific refs (`refs/heads/main`, `refs/tags/v*`) or environments.

3. **Trust policy `sub: repo:org/repo:pull_request` -> the SAME `sub` for every PR.** Including PRs from forks. Effectively assumable by any contributor. Never scope production roles to `pull_request`.

4. **Signing a tag instead of a digest -> meaningless signature.** Tags are mutable; the signature is bound to a label that can be re-pointed. Always sign `IMAGE@DIGEST`, never `IMAGE:TAG`. Use `steps.build.outputs.digest`.

5. **GHA cache 10 GB shared across all caches in repo -> images evict each other in monorepos.** Use `scope=<image-name>` per build. Without it, the cache is a single shared pool and you cache-hit zero of your images.

6. **`docker buildx build --load` strips BuildKit attestations.** The local docker daemon doesn't support multi-manifest layouts. Attestations only attach when pushing to a registry. If you need a local image for testing, build twice (or `--push` to a private throwaway registry).

7. **Trivy DB re-download (~600 MB) on every run.** Set `cache-dir: ~/.cache/trivy` and pair with `actions/cache`. Saves ~30-90 seconds per scan.

8. **`github/codeql-action/upload-sarif@v3` -> outdated.** v4 is the current major (v4.35.3 May 2026). v3 still receives parallel patches but new pipelines should target v4. Specifically called out as a copy-paste foot-gun.

9. **Third-party action not SHA-pinned in production.** Tj-actions / CVE-2025-30066 (March 2025) is the canonical incident -- a compromised version of `tj-actions/changed-files` would have leaked secrets from every consuming workflow. Float majors (`@v4`) only while learning; SHA-pin everything in real orgs.

10. **`role-session-name` lacking `${{ github.run_id }}` -> CloudTrail forensics is impossible.** When you need to trace "which CI run did this?", the session name is the only handle. Always include the run ID and run attempt.

11. **`--certificate-identity-regexp` without `^...$` anchors -> prefix-match attack.** `https://github.com/my-org/` matches `https://github.com/my-org-evil-twin/...`. Always anchor regex policies.

12. **Long-lived `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` in GitHub secrets.** They leak in workflow logs (forgotten `::add-mask::`), rotate poorly (manual), and grant the same blast radius to every workflow. Switch to OIDC; this is non-negotiable in 2026.

13. **`docker/setup-buildx-action@v4` is the current major (Mar 2026).** Older blog posts pin `@v3`. Upgrade -- v4 enables the latest BuildKit features and the v2 GHA cache backend defaults.

14. **`pull_request_target` workflows that check out PR code AND have access to secrets.** This is the OWASP Top-10 of GitHub Actions; the entire class of "fork PR runs your workflow with your secrets" attacks lives here. `pull_request_target` is for trusted-base-context workflows only (label triage, comment automation); never check out PR code in one.

15. **`provenance: true` on a registry that doesn't support OCI v1.1 multi-manifest** (old Artifactory, very old Harbor, ECR Public until mid-2024) -> push fails with `unsupported media type` or silently strips the attestation. Verify your registry supports it before relying on it; ECR Private and GHCR are safe.

---

## Part 12: Decision Frameworks

### Framework 1: Cache backend choice

| Question | `type=gha` | `type=registry` | `type=inline` |
|---|---|---|---|
| Default? | YES | When >2 GB or monorepo | Rarely |
| Image size | < 2 GB | Any | Bloats image |
| Cross-fork share? | NO | YES | YES |
| Supports `mode=max`? | YES | YES | NO |
| Cost | Free (10 GB/repo) | Storage cost | Free (image bloat) |

### Framework 2: Trivy gating per trigger

| Trigger | Severity gate | Hard fail? | SARIF upload |
|---|---|---|---|
| `push: main, tags: v*` (release) | CRITICAL,HIGH | YES | YES |
| `pull_request` (app code) | CRITICAL,HIGH | Depends on policy | YES |
| `pull_request` (docs only) | n/a | n/a | n/a (path-filtered out) |
| `nightly schedule` | LOW,MEDIUM,HIGH,CRITICAL | NO | YES |

### Framework 3: Tag-strategy by environment

| Environment | Reference style | Why |
|---|---|---|
| Production K8s manifest | `image@sha256:abc...` (digest) | Immutable; signature-verifiable |
| Staging K8s manifest | `image:1.2.3` (full semver) | Immutable in practice; human-readable |
| Dev K8s manifest | `image:main` (branch tag) | Auto-updates with each main push |
| Local dev `docker run` | `image:latest` | Convenience; no traceability needed |
| Build dashboards | `image:sha-abc1234` (short SHA) | Compact; matches commit |

### Framework 4: Registry choice

| Use case | Registry |
|---|---|
| Production images deployed to AWS (EKS, ECS, Lambda) | ECR |
| Open-source images for community consumption | GHCR (free public) or Docker Hub |
| Internal images shared across many GitHub repos | GHCR (org-scoped, OIDC-friendly) |
| Multi-cloud / multi-platform pulls | Docker Hub (CDN reach) or registry.k8s.io |
| Air-gapped environments | Self-hosted Harbor / Artifactory |

### Framework 5: Sign by digest vs sign by tag

There is no decision -- **always sign by digest, never by tag.**

| Aspect | Digest | Tag |
|---|---|---|
| Mutability | Immutable (cryptographic) | Mutable (re-pointable) |
| Signature meaning | "These exact bytes are signed" | "Whoever owns this label is signed" (uselessly weak) |
| Verifiable at admission | YES | YES, but signature can be invalidated by a tag re-point |
| Recommendation | ALWAYS | NEVER |

The only reason this is a "framework" is because the wrong choice (sign by tag) is the most common authoring mistake.

---

## Part 13: Verification Cheat Sheet -- CLI Commands

Useful for debugging a workflow OR for validating an image consumed downstream.

```bash
# ------ AWS / OIDC sanity ------

# Verify OIDC actually worked in a workflow step
aws sts get-caller-identity

# List the IAM identity provider in an account
aws iam list-open-id-connect-providers

# Inspect the trust policy of a role
aws iam get-role --role-name GHA-MyRepo-DeployRole \
  --query 'Role.AssumeRolePolicyDocument'


# ------ ECR / image inspection ------

# Login to ECR locally for testing
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin 123.dkr.ecr.us-east-1.amazonaws.com

# Inspect an image's manifest list (shows all platforms + attestations)
docker buildx imagetools inspect 123.dkr.ecr.us-east-1.amazonaws.com/api:1.2.3

# Pretty-print the SBOM
docker buildx imagetools inspect 123.dkr.ecr.us-east-1.amazonaws.com/api:1.2.3 \
  --format '{{ json .SBOM }}' | jq

# Pretty-print the provenance
docker buildx imagetools inspect 123.dkr.ecr.us-east-1.amazonaws.com/api:1.2.3 \
  --format '{{ json .Provenance }}' | jq


# ------ Trivy ------

# Local scan equivalent to the CI gate
trivy image \
  --severity CRITICAL,HIGH \
  --exit-code 1 \
  --ignore-unfixed \
  --scanners vuln,secret \
  123.dkr.ecr.us-east-1.amazonaws.com/api:1.2.3


# ------ Cosign ------

# Verify a signature against an identity policy
cosign verify \
  --certificate-identity-regexp '^https://github.com/my-org/[^/]+/\.github/workflows/release\.yml@refs/heads/main$' \
  --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' \
  '123.dkr.ecr.us-east-1.amazonaws.com/api@sha256:abc...'

# Get the Rekor log entry for a signature
cosign verify --output text ... \
  | grep 'tlog' || rekor-cli search --sha sha256:abc...

# Inspect the certificate that signed the image
cosign verify ... --output-file - 2>/dev/null \
  | jq '.[0].optional.Bundle.Payload' | base64 -d | jq
```

---

## Part 14: 2026 Currency Notes

- **`docker/setup-buildx-action@v4`** is the current major (Mar 2026). Pin `@v4` (or SHA). v3 was current before; older blog posts often still cite it.
- **`docker/build-push-action@v7`** (Mar 2026) -- v7 introduced default attestation behavior changes (now opt-out for some image types when `provenance` is set explicitly).
- **`docker/metadata-action@v6`** (Mar 2026) -- v6 dropped some legacy tag prefixes; if migrating from v5, double-check your `tags:` block.
- **`aws-actions/configure-aws-credentials@v6`** (Feb 2026) -- v6 made session tagging the default, deprecated some legacy `aws-access-key-id` / `aws-secret-access-key` input modes (still work, but warned).
- **`sigstore/cosign-installer@v4`** (May 2026) -- v4 dropped Node 16 support. v3 still works on legacy runners but is end-of-life.
- **`github/codeql-action/upload-sarif@v4`** (May 2026, v4.35.3) -- v4 is current. v3 still receives patches in parallel for orgs with slow upgrade cadences. **New pipelines must target v4.**
- **`actions/attest-build-provenance@v4`** (Feb 2026) -- GitHub-native attestation alternative; complements (doesn't replace) BuildKit's `provenance: true`.
- **OIDC IAM provider thumbprints**: AWS no longer enforces thumbprint validation server-side as of 2023. Older Terraform modules may still demand a specific thumbprint -- pass `["ffffffffffffffffffffffffffffffffffffffff"]` (40 f's) if your module requires the parameter but you don't want to track the actual thumbprint.
- **GHA cache v1 EOL'd in early 2025.** `setup-buildx-action@v4` automatically configures the v2 backend; nothing to do unless you have a manual cache config from before then.
- **ECR Public** supported OCI v1.1 multi-manifest (and therefore BuildKit attestations) starting mid-2024. ECR Private has supported it since 2023.

---

## The 10 Commandments of Container CI

1. **Sign by digest, never by tag.** The seal goes on the loaf, not on the sticker. Use `steps.build.outputs.digest` as the cosign target, every time.

2. **Use OIDC; never store long-lived AWS keys in GitHub secrets.** `permissions: id-token: write` + `aws-actions/configure-aws-credentials@v6` + a tightly-scoped trust policy. No exceptions in 2026.

3. **Scope the trust policy `sub` to a specific ref or environment.** `repo:org/repo:ref:refs/heads/main` is the production default. `repo:org/repo:pull_request` is a privilege-escalation foot-gun. `repo:org/repo:*` is the same foot-gun, doubled.

4. **Memorize the four-action stack as one block.** `setup-qemu` -> `setup-buildx` -> `login` -> `build-push`. The order is causal, not stylistic.

5. **Scope the BuildKit cache per-image** (`scope=api`, `scope=worker`). Without `scope=`, monorepo images evict each other and your "cached" pipeline silently runs cold.

6. **Use `docker/metadata-action@v6` for tag generation.** The semver trio (`{{version}}`, `{{major}}.{{minor}}`, `{{major}}`) plus `type=raw,value=latest,enable={{is_default_branch}}` covers 95% of needs; never write tag-generation bash by hand.

7. **Two-tier Trivy gate**: SARIF upload for everything, hard fail on CRITICAL/HIGH (`exit-code: 1` + `ignore-unfixed: true`). Use `github/codeql-action/upload-sarif@v4` (NOT v3). `.trivyignore` for accepted CVEs with comments and review dates.

8. **Always emit BuildKit attestations** in production (`provenance: mode=max` + `sbom: true`). Free SLSA Level 3 evidence; travels with the image; verifiable via `docker buildx imagetools inspect`.

9. **Anchor `--certificate-identity-regexp` with `^...$`.** Prefix matching is a real attack vector; `^https://github.com/my-org/` matches `^https://github.com/my-org-evil-twin/...`. Always anchor.

10. **SHA-pin third-party actions in production.** Float `@v4` only while learning. Tj-actions / CVE-2025-30066 is the canonical "we floated and it bit us" lesson; Dependabot can SHA-bump for you.

---

## Cheat Sheet -- Action Versions and Key Flags

```yaml
# ============================================================================
# CONTAINER CI -- THE FLAGSHIP RECIPE (May 2026 versions)
# ============================================================================

permissions:
  contents: read
  id-token: write       # OIDC for AWS + Cosign keyless
  packages: write       # GHCR push (omit if ECR-only)
  security-events: write # SARIF upload to Code Scanning

steps:
  # 1. Source
  - uses: actions/checkout@<sha>                          # v4.2.2

  # 2. Build infrastructure
  - uses: docker/setup-qemu-action@v4                     # v4.0.0 (Mar 2026)
  - uses: docker/setup-buildx-action@v4                   # v4.0.0 (Mar 2026)

  # 3. Auth
  - uses: aws-actions/configure-aws-credentials@v6        # v6.1.1 (May 2026)
    with:
      role-to-assume: arn:aws:iam::ACCT:role/GHA-...
      role-session-name: gha-${{ github.run_id }}
      aws-region: us-east-1
  - run: aws sts get-caller-identity                      # OIDC sanity check
  - id: ecr-login
    uses: aws-actions/amazon-ecr-login@v2                 # v2.1.5 (May 2026)

  # 4. Tags + labels
  - id: meta
    uses: docker/metadata-action@v6                       # v6.0.0 (Mar 2026)
    with:
      images: ${{ steps.ecr-login.outputs.registry }}/api
      tags: |
        type=sha,prefix=,format=long
        type=sha,format=short
        type=ref,event=branch
        type=ref,event=pr
        type=semver,pattern={{version}}
        type=semver,pattern={{major}}.{{minor}}
        type=semver,pattern={{major}}
        type=raw,value=latest,enable={{is_default_branch}}

  # 5. Build + push (with attestations + cache)
  - id: build
    uses: docker/build-push-action@v7                     # v7.1.0 (Apr 2026)
    with:
      context: .
      push: true
      tags: ${{ steps.meta.outputs.tags }}
      labels: ${{ steps.meta.outputs.labels }}
      cache-from: type=gha,scope=api
      cache-to:   type=gha,mode=max,scope=api
      provenance: mode=max
      sbom: true

  # 6. Scan -- two-tier gate. First pass: report-only SARIF for visibility.
  - uses: aquasecurity/trivy-action@v0.36.0               # v0.36.0 (Apr 2026)
    with:
      image-ref: ${{ steps.ecr-login.outputs.registry }}/api@${{ steps.build.outputs.digest }}
      severity: 'UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL'
      exit-code: '0'
      format: 'sarif'
      output: 'trivy-results.sarif'
      ignore-unfixed: true
      cache-dir: ~/.cache/trivy
  - if: always()
    uses: github/codeql-action/upload-sarif@v4            # v4.35.3 (May 2026 — NOT v3)
    with: { sarif_file: trivy-results.sarif }
  # Second pass: hard gate -- block on CRITICAL/HIGH only.
  - uses: aquasecurity/trivy-action@v0.36.0
    with:
      image-ref: ${{ steps.ecr-login.outputs.registry }}/api@${{ steps.build.outputs.digest }}
      severity: 'CRITICAL,HIGH'
      exit-code: '1'
      ignore-unfixed: true
      cache-dir: ~/.cache/trivy

  # 7. Sign
  - uses: sigstore/cosign-installer@v4                    # v4.1.2 (May 2026)
  - env:
      COSIGN_YES: 'true'
      IMAGE: ${{ steps.ecr-login.outputs.registry }}/api@${{ steps.build.outputs.digest }}
    run: cosign sign --yes "${IMAGE}"

  # 8. (Optional) GitHub-native provenance attestation
  # - uses: actions/attest-build-provenance@v4            # v4.1.0 (Feb 2026)
  #   with:
  #     subject-name: ${{ steps.ecr-login.outputs.registry }}/api
  #     subject-digest: ${{ steps.build.outputs.digest }}
  #     push-to-registry: true
```

```bash
# ============================================================================
# VERIFICATION COMMANDS
# ============================================================================

# OIDC sanity
aws sts get-caller-identity

# Image manifest, SBOM, provenance
docker buildx imagetools inspect <image>:<tag>
docker buildx imagetools inspect <image>:<tag> --format '{{ json .SBOM }}'
docker buildx imagetools inspect <image>:<tag> --format '{{ json .Provenance }}'

# Local Trivy run (matches CI gate)
trivy image --severity CRITICAL,HIGH --exit-code 1 --ignore-unfixed <image>:<tag>

# Cosign verify (admission-controller style)
cosign verify \
  --certificate-identity-regexp '^https://github.com/my-org/[^/]+/\.github/workflows/release\.yml@refs/heads/main$' \
  --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' \
  '<image>@<digest>'
```

```json
// ============================================================================
// IAM TRUST POLICY (the security spine)
// ============================================================================
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

---

## Further Reading

- [Docker -- GitHub Actions quickstart](https://docs.docker.com/build/ci/github-actions/) -- the canonical four-action stack
- [Docker -- Cache management for GitHub Actions](https://docs.docker.com/build/ci/github-actions/cache/) -- `type=gha` vs `type=registry` vs `type=inline`
- [docker/metadata-action README](https://github.com/docker/metadata-action) -- exhaustive tag patterns
- [GitHub Docs -- Configuring OIDC in AWS](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws) -- the security spine
- [AWS Security Blog -- Use IAM roles to connect GitHub Actions to AWS](https://aws.amazon.com/blogs/security/use-iam-roles-to-connect-github-actions-to-actions-in-aws/) -- the "why" behind OIDC
- [Sigstore Cosign quickstart](https://docs.sigstore.dev/quickstart/quickstart-cosign/) -- keyless signing flow
- [Aqua Security -- Trivy GitHub Action](https://github.com/aquasecurity/trivy-action) -- scanning patterns
- [Docker -- SBOM attestations with BuildKit](https://docs.docker.com/build/metadata/attestations/sbom/) -- in-toto attestations
- [SLSA spec v1.0](https://slsa.dev/spec/v1.0/levels) -- the framework BuildKit provenance maps to
- Yesterday's doc: [GitHub Actions Custom Actions & Reusable Patterns](../cicd/2026-05-06-github-actions-custom-actions-reusable-patterns.md)
- Two days ago: [GitHub Actions Core Review & Advanced Workflows](../cicd/2026-05-05-github-actions-advanced-workflows.md)
- Related: [ArgoCD Image Updater](../../april/argocd/) -- how the digest output flows into GitOps reconciliation
