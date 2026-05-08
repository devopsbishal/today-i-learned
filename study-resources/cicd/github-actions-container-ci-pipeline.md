# GitHub Actions -- Container CI: Build, Scan, Sign & Push -- 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-05-07
**Last verified:** 2026-05-07 (action versions checked against GitHub Releases API)
**Difficulty:** Intermediate -> Advanced (assumes solid Docker multi-stage / BuildKit knowledge from the prior 7 Docker docs and the Week 6.8 GitHub Actions core + custom-actions plans; this plan stitches those two halves together into a production container CI pipeline -- build with cache, tag deterministically, scan, sign, push -- with OIDC as the security spine)

## Overview

This plan covers the full container CI cutting line that real teams ship: `docker/build-push-action` with multi-stage builds and the GHA cache backend, deterministic tagging via `docker/metadata-action` (SHA + semver + branch), Trivy vulnerability scanning with severity-based gating, BuildKit-native SBOM/provenance attestations, Cosign keyless signing using GitHub's OIDC token, and pushing to ECR/GHCR -- with **AWS OIDC federation (no long-lived access keys)** as the security thread that ties it together. After completing it, you should be able to author a single `release.yml` that builds an image once, tags it five different ways, fails the build on a CRITICAL CVE, generates an SBOM and SLSA provenance attestation, signs the image keylessly with Sigstore, pushes to ECR, and never touches a long-lived `AWS_SECRET_ACCESS_KEY` -- and you should be able to defend each design choice in a code review.

## Action Versions Reference (verified 2026-05-07)

> Many third-party tutorials online still cite the previous majors. Use this table when you copy-paste examples -- if a snippet pins to an older major, upgrade it. Always SHA-pin in production CI for security; the major-version float is OK while learning.

| Action | Current major | Latest patch | Notes |
| --- | --- | --- | --- |
| `docker/setup-qemu-action` | **v4** | v4.0.0 | Major bumped Mar 2026 |
| `docker/setup-buildx-action` | **v4** | v4.0.0 | Major bumped Mar 2026 |
| `docker/login-action` | **v4** | v4.1.0 | Major bumped Mar 2026 |
| `docker/metadata-action` | **v6** | v6.0.0 | Major bumped Mar 2026 |
| `docker/build-push-action` | **v7** | v7.1.0 | Major bumped Mar 2026 |
| `aws-actions/configure-aws-credentials` | **v6** | v6.1.1 | Major bumped Feb 2026 |
| `aws-actions/amazon-ecr-login` | v2 | v2.1.5 | v2 still current |
| `aquasecurity/trivy-action` | v0.x | v0.36.0 | Pre-1.0, pin to a specific tag |
| `sigstore/cosign-installer` | **v4** | v4.1.2 | Major bumped 2026 |
| `github/codeql-action/upload-sarif` | **v4** | v4.35.3 | v4 is the current major; v3 still patched in parallel |
| `actions/attest-build-provenance` | **v4** | v4.1.0 | Major bumped Feb 2026 |

## Resources

### 1. Docker Build GitHub Actions -- Official Quickstart (Docker Docs) -- 12 min -- MUST READ

- **URL:** <https://docs.docker.com/build/ci/github-actions/>
- **Type:** Official Docs (quickstart)
- **Summary:** The canonical entry point that wires the four-action stack you will use every time: `docker/setup-qemu-action@v4` (multi-arch emulation, only needed when building `linux/arm64` from an `amd64` runner), `docker/setup-buildx-action@v4` (boots a BuildKit builder with the v2 GHA cache backend pre-configured -- the action sets the `url` and `token` parameters automatically as of early 2025 when v1 was shut down), `docker/login-action@v4` (registry auth -- swap `username/password` for ECR via `aws-actions/amazon-ecr-login` outputs), and `docker/build-push-action@v7` (the actual build + push -- v7 was released March 2026 and is the current major). Skim the matrix-build, multi-platform, and build-args sections -- you will reference these patterns daily.

### 2. Cache management with GitHub Actions (Docker Docs) -- 15 min -- MUST READ

- **URL:** <https://docs.docker.com/build/ci/github-actions/cache/>
- **Type:** Official Docs (deep dive)
- **Summary:** The three cache backends you must know: (a) **`type=gha`** -- BuildKit cache stored in the GitHub Actions cache (10 GB repo limit shared with `actions/cache`, scoped per-ref by default, `mode=max` to cache *all* layers including intermediate stages vs `mode=min` for only the final stage); (b) **`type=registry`** -- cache pushed to a separate registry tag (e.g. `myrepo:buildcache`), unlimited size, survives across forks/branches, the right choice for monorepos with images >2 GB; (c) **`type=inline`** -- cache embedded into the image manifest itself, simplest setup but only works with `mode=min` and bloats the image. The `cache-from`/`cache-to` syntax accepts multiple sources (`cache-from: type=gha\ntype=registry,ref=myrepo:cache`) so you can read from both and write to one. Critical gotcha: **changing `cache-to` does not invalidate `cache-from`** -- a stale entry will still be pulled until the cache expires (7 days idle for GHA cache).

### 3. docker/metadata-action@v6 README -- Tags & Labels (GitHub) -- 12 min -- MUST READ

- **URL:** <https://github.com/docker/metadata-action>
- **Type:** Official Action README
- **Summary:** The action that turns "what tags should this image have?" from 30 lines of bash into 8 lines of YAML. v6 was released March 2026 -- pin to `@v6` (or a SHA). Tag types you will use: `type=sha,prefix=,format=long` (immutable 40-char SHA -- the only tag that should ever appear in a Kubernetes manifest), `type=sha,format=short` (12-char `sha-abc1234` for human-readable build artifacts), `type=ref,event=branch` (`main`, `feature-foo` -- mutable, fine for dev), `type=ref,event=pr` (`pr-123` -- ephemeral, perfect for review apps), `type=semver,pattern={{version}}` + `type=semver,pattern={{major}}.{{minor}}` + `type=semver,pattern={{major}}` (the trio that lets consumers pin to `v1`, `v1.2`, or `v1.2.3` from a single git tag push), and `type=raw,value=latest,enable={{is_default_branch}}` (the conditional `latest` you almost always want). Outputs `tags` (newline-separated, fed straight into `docker/build-push-action`'s `tags:` input) and `labels` (OCI image labels including `org.opencontainers.image.source` -- this is what makes "View source" work on GHCR package pages).

### 4. Configuring OpenID Connect in AWS (GitHub Docs) -- 20 min -- MUST READ -- THE security topic

- **URL:** <https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws>
- **Type:** Official Docs
- **Summary:** The end-to-end blueprint for replacing `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` GitHub secrets (which are long-lived, leak in logs, and rotate poorly) with short-lived STS credentials minted per workflow run. The four moving parts: (1) Create an **OIDC identity provider** in IAM pointing at `https://token.actions.githubusercontent.com` with audience `sts.amazonaws.com` (one-time per AWS account); (2) Create an **IAM role** with a trust policy whose `Principal` is that OIDC provider and whose `Condition` block evaluates `token.actions.githubusercontent.com:aud` (= `sts.amazonaws.com`) AND `token.actions.githubusercontent.com:sub` (= `repo:my-org/my-repo:ref:refs/heads/main` OR `repo:my-org/my-repo:environment:production` -- this is the line that prevents *any other repo's* workflow from assuming your role); (3) Add `permissions: id-token: write` at the workflow or job level (without this the OIDC token is not minted -- `Error: Could not load credentials from any providers` is the classic symptom); (4) Use `aws-actions/configure-aws-credentials@v4` with `role-to-assume: arn:aws:iam::<account>:role/<role-name>` and `aws-region:` -- the action calls `sts:AssumeRoleWithWebIdentity` and exports the temp creds as env vars for downstream steps. The `sub` claim format is the most error-prone part: branch-scoped vs environment-scoped vs tag-scoped trust each have different `sub` strings, and a wildcard like `repo:my-org/my-repo:*` opens the role to forks and pull requests -- almost never what you want.

### 5. aws-actions/configure-aws-credentials@v6 README (GitHub) -- 10 min -- MUST READ

- **URL:** <https://github.com/aws-actions/configure-aws-credentials>
- **Type:** Official Action README
- **Summary:** The action that performs the OIDC handshake from #4. **v6 is the current major (released Feb 2026, latest patch v6.1.1 May 2026)** -- if you're copy-pasting from older blog posts pinned to `@v4`, upgrade. Read these sections: **OIDC** (the `role-to-assume` + `aws-region` happy path, plus `role-session-name` for CloudTrail attribution -- use `GitHubActions-${{ github.run_id }}` so each run is auditable), **Session tagging** (passes `repository`, `workflow`, `actor`, etc. as STS session tags so you can write IAM policies like `Condition: StringEquals: aws:RequestTag/repository: my-org/my-repo` -- defense-in-depth on top of the trust policy), **role-chaining** (when a single workflow needs to assume multiple roles -- e.g. dev account -> prod account -- call the action twice with `role-chaining: true` on the second invocation), and **Credentials lifetime** (`role-duration-seconds` defaults to 1 hour, max varies by role's `MaxSessionDuration`; for long workflows set this explicitly). Pair with `aws-actions/amazon-ecr-login@v2` (still the current major, latest v2.1.5) -- it consumes the temp creds and emits a `registry` output (`123456.dkr.ecr.us-east-1.amazonaws.com`) that you feed into `docker/login-action`'s `registry:` input.

### 6. aquasecurity/trivy-action README -- Severity Gating in CI (GitHub) -- 12 min

> Pin to `@v0.36.0` (or later -- this action is still pre-1.0 so the major hasn't bumped; latest is v0.36.0 from April 2026).

- **URL:** <https://github.com/aquasecurity/trivy-action>
- **Type:** Official Action README
- **Summary:** The most common open-source container vulnerability scanner integrated into CI. Three production patterns: (a) **Single hard gate** -- `severity: 'CRITICAL,HIGH'` + `exit-code: '1'` + `ignore-unfixed: true` (don't fail on CVEs that have no patch available, you cannot act on them anyway) -- the minimum useful gate; (b) **Two-tier gate** -- run the action twice: first with `severity: 'UNKNOWN,LOW,MEDIUM'` + `exit-code: '0'` + `format: 'sarif'` + `output: 'trivy-results.sarif'` to upload findings to GitHub Code Scanning (visible in the Security tab) without failing the build, then again with `severity: 'CRITICAL,HIGH'` + `exit-code: '1'` to actually block the merge; (c) **Allowlisted CVEs** -- `.trivyignore` file at repo root listing CVE IDs the team has accepted with comments and review dates. Critical flags: `vuln-type: 'os,library'` (OS packages + app deps -- the default), `scanners: 'vuln,secret,misconfig'` (Trivy can also detect leaked secrets in the image and Dockerfile misconfigs), and `cache-dir:` (cache the vuln DB across runs, otherwise every scan re-downloads ~600 MB of CVE feeds). Pair with `github/codeql-action/upload-sarif@v4` (v4 became the current major in 2026 -- latest v4.35.3 May 2026; v3 still receives parallel patches but new pipelines should be on v4) to push the SARIF into Code Scanning -- the result is annotated PR diffs showing exactly which line of which Dockerfile introduced a vulnerable package.

### 7. SBOM Attestations with BuildKit (Docker Docs) -- 10 min

- **URL:** <https://docs.docker.com/build/metadata/attestations/sbom/>
- **Type:** Official Docs
- **Summary:** Why standalone `syft` invocations are no longer the right answer. BuildKit's attestation support originally landed in 0.11 (shipped via `docker/setup-buildx-action@v3`) and is now stable in `docker/setup-buildx-action@v4` -- passing `provenance: true` and `sbom: true` (or the unified `attests: type=sbom,type=provenance,mode=max`) to `docker/build-push-action@v7` produces an in-toto attestation **bundled into the OCI image manifest** under a separate manifest with `application/vnd.in-toto+json` media type -- one image artifact, one digest, with the SBOM and SLSA provenance attached. SBOM format is SPDX JSON, generator is Anchore Syft (swappable via the `generator:` parameter). Critical caveats: (a) the attestation only attaches when pushing to a **registry** -- `docker buildx build --load` to local docker daemon strips it because the local daemon does not understand the OCI image index v1.1 format; (b) some older registries (Artifactory <7.59, ECR Public until mid-2024) reject the multi-manifest layout -- ECR Private and GHCR support it natively; (c) verifying the SBOM at consumption time uses `docker buildx imagetools inspect <image> --format '{{ json .SBOM }}'`. This is the modern replacement for "run syft in a separate step and upload the JSON as an artifact" -- it travels *with* the image.

### 8. Sigstore Quickstart with Cosign -- Keyless Signing in GitHub Actions (Sigstore Docs) -- 15 min -- MUST READ

- **URL:** <https://docs.sigstore.dev/quickstart/quickstart-cosign/>
- **Type:** Official Docs
- **Summary:** Keyless signing is what makes "I built and signed this image" verifiable without managing GPG keys or HSMs. The flow: (1) install Cosign in the workflow via `sigstore/cosign-installer@v4` (v4 is the current major; latest v4.1.2 May 2026 -- the v3 -> v4 bump dropped Node 16, so v4 is required on modern runners); (2) request `permissions: id-token: write` (same OIDC token used for AWS); (3) run `cosign sign --yes <image>@<digest>` -- Cosign uses the GitHub-issued OIDC token to authenticate to **Fulcio** (Sigstore's CA), Fulcio mints a short-lived (10-minute) X.509 cert binding the workflow identity (`repo:my-org/my-repo:ref:refs/heads/main`) to an ephemeral signing key, Cosign signs the image digest, the signature + cert are pushed to the registry alongside the image as a `.sig` tag, and the entire transaction is logged to **Rekor** (the public transparency log) so anyone can independently verify the signature was minted at a specific time by a specific workflow. Verify with `cosign verify <image> --certificate-identity-regexp '^https://github.com/my-org/' --certificate-oidc-issuer https://token.actions.githubusercontent.com` -- the `--certificate-identity` flag is the policy gate (e.g., admission controller in Kubernetes only allows images signed by `^https://github.com/my-org/.*` workflows on `refs/heads/main`). Always sign by **digest** (`@sha256:...`), never by tag -- signing a mutable tag is meaningless because the tag can be repointed.

### 9. AWS Security Blog -- Use IAM Roles to Connect GitHub Actions to AWS (AWS Blog) -- 10 min

- **URL:** <https://aws.amazon.com/blogs/security/use-iam-roles-to-connect-github-actions-to-actions-in-aws/>
- **Type:** AWS Official Blog
- **Summary:** The "why" article behind resource #4 -- AWS's own security-team explanation of why long-lived access keys in GitHub secrets are an anti-pattern (they appear in every workflow run's env, leak in logs if you forget `::add-mask::`, lack rotation hygiene, and grant the same blast radius regardless of which workflow uses them) and how OIDC federation fixes each of those. Pay attention to the `sub` claim conditioning examples -- particularly the `repo:org/repo:environment:production` pattern, which combines GitHub Environments' protection rules (manual approvers, branch restrictions, wait timers) with AWS IAM (so a role can only be assumed when a workflow is running against a protected environment that already enforces those gates). Also covers the trust-policy gotcha around `pull_request` triggers -- by default `sub` for a PR is `repo:org/repo:pull_request`, which is *the same* for every PR including from forks, so a role scoped to that `sub` is effectively assumable by any contributor's PR. Solution: scope to `ref:refs/heads/main` (only main-branch pushes can deploy) or to environments (with required reviewers).

### 10. (Nice-to-have) Building a Secure Container Pipeline -- SBOM, Signing, Attestation (Nine Lives Blog) -- 8 min

- **URL:** <https://nineliveszerotrust.com/blog/container-sbom-signing-attestation/>
- **Type:** Practitioner Blog (end-to-end example)
- **Summary:** A complete `release.yml` that stitches all of the above into one workflow: build with BuildKit attestations, scan with Trivy gated on HIGH/CRITICAL, push to GHCR, then `cosign sign` + `cosign attest --predicate sbom.spdx.json --type spdx` (the second `attest` adds the SBOM as a signed *attestation* on top of the BuildKit-attached SBOM -- belt-and-suspenders for compliance use cases like FedRAMP / SLSA Level 3). Useful as a reference layout when assembling your own workflow file -- compare its job ordering (build -> scan -> sign in series, never parallel because scan and sign both consume the digest the build emits) and its use of `outputs.digest` from `docker/build-push-action` (the immutable handle you sign and verify against) vs `outputs.imageid` (which is a local-only, non-pullable identifier and should never appear in a `cosign verify` command).

## Study Tips

- **Pin the four-action stack in your head as one block.** `setup-qemu-action` -> `setup-buildx-action` -> `login-action` -> `build-push-action`. The first two only need `with:` if you want a non-default builder; the third needs registry-specific auth (ECR via `amazon-ecr-login` outputs, GHCR via `${{ secrets.GITHUB_TOKEN }}` with `permissions: packages: write`, DockerHub via PAT secret); the fourth is where 90% of your YAML goes (`tags:`, `labels:`, `cache-from`, `cache-to`, `provenance:`, `sbom:`, `platforms:`). Memorizing this skeleton is more valuable than memorizing flags.
- **Build the OIDC trust policy on a whiteboard before writing IAM JSON.** Three questions: (1) Which AWS account is the role in? (2) Which exact `sub` claim should be allowed -- `repo:org/repo:ref:refs/heads/main` (single branch), `repo:org/repo:environment:production` (any branch, but only when running against a protected environment), `repo:org/repo:pull_request` (DANGER -- any PR including forks), or `repo:org/repo:*` (DOUBLE DANGER -- any workflow on any ref)? (3) What's the `aud` -- always `sts.amazonaws.com` for the official action unless you customized it. Get those three right and the JSON writes itself; get the `sub` wrong and you have a privilege-escalation foot-gun.
- **Always sign and verify by digest, never by tag.** `docker/build-push-action` exposes the pushed digest as `steps.<id>.outputs.digest`. Pass `${{ steps.build.outputs.digest }}` (a `sha256:...` string) into your `cosign sign` and ECS/EKS deployment manifests. Tags are mutable; a digest is the cryptographic identity of the bytes. This is the same discipline you applied to pinning third-party actions to SHAs in yesterday's plan.
- **Treat Trivy's exit code as a policy decision, not a default.** `exit-code: 1` on CRITICAL is the right default for `release` workflows that build images consumed in production; on `pull_request` workflows, you may want `exit-code: 0` + SARIF upload so reviewers see findings inline without forcing a force-push to fix a brand-new CVE in a transitive dep that landed an hour ago. Decide per-trigger.
- **Cache scope is the most-overlooked dial.** `type=gha` shares a 10 GB pool with all your other workflows' `actions/cache` entries. If you build 4 different images per repo, scope each: `cache-to: type=gha,mode=max,scope=api` / `scope=worker` / `scope=migrator` -- otherwise each build evicts the previous one's cache and your "cache-enabled" pipeline silently runs cold every time.

## Next Steps

- **Day 4 (Security Hardening for GHA Workflows)** -- the broader `permissions:` minimization story (default `GITHUB_TOKEN` to `contents: read`, grant `packages: write` only on the build job, `id-token: write` only on the deploy job), `pull_request_target` foot-guns, third-party action SHA pinning + Dependabot, and the `step-security/harden-runner` action (egress allow-listing for runners -- prevents a compromised dependency from exfiltrating secrets to an attacker-controlled domain).
- **Tying it back to GitOps** -- the production pattern is `release.yml` (this plan: build/scan/sign/push image to ECR) -> `update-manifests.yml` (clone the config repo, bump the image digest in `values.yaml`, open a PR back) -> ArgoCD Image Updater (covered in Apr 17-19 docs) reconciles the new digest. The signed attestation from this plan is what an admission controller (Kyverno, Gatekeeper, or Sigstore Policy Controller) verifies before allowing the pod to start.
- **SLSA Levels & supply-chain attestations** -- the `provenance: true` flag in `build-push-action` emits SLSA v0.2 provenance; combined with Cosign keyless signing on Rekor, this gets you to SLSA Level 3 (verifiable, non-falsifiable build provenance from a hardened build platform). Read the SLSA spec (`https://slsa.dev/spec/v1.0/levels`) and the GitHub `actions/attest-build-provenance@v4` action (v4.1.0 Feb 2026 is current) which writes to GitHub's own attestation API as an alternative to Rekor.
- **Multi-arch builds at scale** -- the `platforms: linux/amd64,linux/arm64` flag with QEMU is *correct* but slow (ARM64 emulated on amd64 runners is ~5x slower). For production, run a matrix across `runs-on: ubuntu-24.04` (amd64) and `runs-on: ubuntu-24.04-arm` (native arm64, GA on GitHub-hosted runners since 2025), then merge the per-arch images into a multi-arch manifest with `docker buildx imagetools create`. Cuts build time roughly in half on dual-arch images.
