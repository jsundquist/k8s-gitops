# Supply chain audit and checklist

Audit date: 2026-11-02. Covers `k8s-gitops`, `knowledgebase`, `sbom`, `transcript-search-mcp` and `jsundquist.github.io` (the blog), across six areas:

1. Artifact hardening
2. Package and dependency management
3. Artifact repositories
4. SBOM
5. SLSA
6. Secure golden paths

Part 1 is what exists today, Part 2 is the per-repo to-do list, Part 3 is the checklist to run on every future repo, Part 4 is the order of work.

**How this was checked.** Workflow, Dependabot and Dockerfile contents were read from `main` on GitHub, not from local checkouts (several are on stale feature branches). Repo settings, branch protection, rulesets, Actions permissions, secrets (names only) and alert counts came from the GitHub API. The published images were inspected with `docker buildx imagetools inspect`.

**Not checked.** GHCR package visibility and settings (the `gh` token lacks `read:packages`; run `gh auth refresh -s read:packages` to include them). Account-level settings such as 2FA, SSH keys and PAT inventory.

Legend: ✅ in place, ⚠️ partial, ❌ missing, ⏳ change open in a pull request, n/a does not apply.

## Progress snapshot (2026-11-02, evening)

Tables in Part 1 show the audit, updated for settings that have since changed on GitHub. Pull requests are not merged until you merge them; check each repo for the current list.

| Repo | Done on `main` | Open PRs | Waiting on you |
|---|---|---|---|
| sbom | SHA pinning required | #31 sign and attest both images, #32 `SECURITY.md`, #33 zizmor bump, #34 and #35 dev-dependency overrides (the 2 Dependabot alerts), #36 base digest bump and drop the `perl-base` apt upgrade | Private vulnerability reporting (off); review #31 before merge, since it changes the release flow |
| jsundquist.github.io | SHA pinning required | #22 Trivy gate, #23 base digest and Docker Dependabot, #24 gitleaks and `pnpm audit`, #25 source SBOM upload, #26 `SECURITY.md`, #27 | Add the `SBOM_DASHBOARD_TOKEN` secret before merging #25; #25 had a failing `actionlint` check when last seen; add `secrets` and `dependency-audit` to required checks after #24 merges |
| knowledgebase | #13 cooldown merged; SHA pinning required; private vulnerability reporting on | #14 security workflow (its `zizmor` fails until the `# main` pin comment in #15 is fixed and #14 is rebased), #15, #16, #17 | Merge in order #15, then rebase and merge #14, then add the four new required checks |
| transcript-search-mcp | CodeQL on; `dependency-review` required; private vulnerability reporting on | #9, #10, #11 (all green), #12 daily SBOM schedule | Decide the auto-merge scope; re-pin the SBOM action when a tag exists |
| k8s-gitops | SHA pinning required; private vulnerability reporting on | this branch: `SECURITY.md`, `rclone` digest pin, CI digest check, docs | Branch protection and the Renovate-versus-Dependabot decision |

---

## Part 1: What you have today

### Scorecard

| Area | k8s-gitops | knowledgebase | sbom | transcript-search-mcp | blog |
|---|---|---|---|---|---|
| 1. Artifact hardening | ⚠️ | n/a (no artifact) | ⚠️ | n/a (no artifact) | ⚠️ |
| 2. Dependency management | ⚠️ | ⚠️ | ✅ | ✅ | ⚠️ |
| 3. Artifact repositories | ❌ | n/a | ⚠️ | n/a | ⚠️ |
| 4. SBOM | ⚠️ | ✅ | ⚠️ | ✅ | ⚠️ |
| 5. SLSA | ❌ | n/a | ❌ | n/a | ❌ |
| 6. Golden path | ⚠️ | ⚠️ | ⚠️ | ⚠️ | ⚠️ |

### Repo settings (verified through the API)

| Setting | k8s-gitops | knowledgebase | sbom | transcript-search-mcp | blog |
|---|---|---|---|---|---|
| Visibility | public | public | public | public | public |
| Secret scanning and push protection | ✅ | ✅ | ✅ | ✅ | ✅ |
| Dependabot security updates | ✅ | ✅ | ✅ | ✅ | ✅ |
| Open Dependabot alerts | 0 | 0 | **2** | 0 | 0 |
| Default workflow token | read | read | read | read | read |
| Actions cannot approve PRs | ✅ | ✅ | ✅ | ✅ | ✅ |
| "Require actions pinned to a full SHA" | ✅ | ✅ | ✅ | ✅ | ✅ |
| Branch protection or ruleset on `main` | **❌ none** | ✅ classic | ✅ ruleset, no bypass actors | ✅ classic | ✅ classic |
| Required status checks | n/a | test x2, lint, audit | test, image-scan, dependency-audit, dependency-review, secrets, workflow-lint, zizmor | lint, build, test, audit, zizmor, secrets, workflow-lint, dependency-review | validate, actionlint, dependency-review, zizmor |
| Required reviews | n/a | none | 0 approvals | 0 approvals | 0 approvals |
| Admins bound by protection | n/a | no | yes (ruleset) | no | no |
| `SECURITY.md` (any location, no shared `jsundquist/.github` repo) | ⏳ this PR | ⏳ PR #16 | ⏳ PR #32 | ⏳ PR #9 | ⏳ PR #26 |
| Private vulnerability reporting | ✅ | ✅ | ❌ off | ✅ | ✅ |
| CodeQL default setup (`code-scanning/default-setup`) | ✅ actions | ✅ actions, python | ✅ actions, JS/TS | ✅ actions, python | ✅ actions, JS/TS |
| Repo secrets | `SBOM_DASHBOARD_TOKEN` | `SBOM_DASHBOARD_TOKEN` | `GITOPS_TOKEN`, `SBOM_DASHBOARD_TOKEN` | `SBOM_DASHBOARD_TOKEN` | `GITOPS_TOKEN` (**no** dashboard token) |

Zero approvals required is reasonable for a solo maintainer, as long as status checks are required.

### Workflow and tooling matrix (on `main`)

| Control | k8s-gitops | knowledgebase | sbom | transcript-search-mcp | blog |
|---|---|---|---|---|---|
| Actions pinned by SHA | ✅ | ✅ | ✅ | ✅ | ✅ |
| `persist-credentials: false` | ✅ | ✅ | ✅ (bump job excepted, documented) | ✅ | ✅ (bump job excepted, documented) |
| Workflow `permissions` least privilege | ✅ | ✅ | ✅ | ✅ | ✅ |
| actionlint | ✅ | ❌ | ✅ | ✅ | ✅ |
| zizmor | ✅ | ❌ | ✅ | ✅ | ✅ |
| gitleaks in CI | ✅ | ❌ | ✅ | ✅ | ❌ |
| gitleaks pre-commit | ✅ lefthook | ❌ | not seen | not seen | not seen |
| dependency-review on PRs | n/a | ❌ | ✅ | ✅ (not a required check) | ✅ |
| Vulnerability audit in CI | n/a | `pip-audit` on push and PR only | `pnpm audit` prod on CI, full weekly | `pip-audit` on CI and weekly | **none** |
| Dependabot cooldown (7 days) | ✅ | ✅ (PR #13) | ✅ | ✅ | ✅ |
| Dependabot ecosystems | actions | uv, actions | npm, actions, docker, docker-compose | uv, actions | npm, actions |
| Lockfile enforced | n/a | `uv sync --frozen` | `--frozen-lockfile` | uv.lock | `--frozen-lockfile` |
| Package manager hardening | n/a | n/a | `allowBuilds`, `minimumReleaseAge` 1 day, audit ignores explained | n/a | `allowBuilds`, overrides; no `minimumReleaseAge` |
| Dependabot auto-merge | no | no | no | yes, **all non-major** deps | yes, **dev deps patch/minor only** |
| Container build in CI | n/a | n/a | ✅ web + worker | n/a | ✅ |
| Trivy scan before push | n/a | n/a | ✅ HIGH/CRITICAL, scanned image is pushed image | n/a | **❌** |
| Base image pinned by digest | n/a | n/a | ✅ | n/a | **❌** (`nginx-unprivileged:1.29-alpine`) |
| Image signed (cosign) | n/a | n/a | ❌ | n/a | ❌ |
| Build provenance attestation | n/a | n/a | ❌ | n/a | ❌ |
| Image SBOM attestation | n/a | n/a | ❌ | n/a | ❌ |
| Source SBOM uploaded to dashboard | n/a (weekly image SBOMs only) | ✅ on push to main | ✅ on push to main | ✅ on push to main | **❌** |
| Weekly image SBOMs to dashboard | ✅ blog, sbom, sbom-worker, postgres, rclone | n/a | covered by k8s-gitops | n/a | covered by k8s-gitops |

### Verified facts about the published images

`docker buildx imagetools inspect` on the currently deployed tags (`sbom:c79019e-4`, `jsundquist.github.io:3c8e15a-47`) returns a plain single-platform Docker v2 manifest for both. There is no OCI index, so **no provenance, no SBOM attestation and no signature are attached to either image**. That makes areas 1 (signing) and 5 (SLSA) a clean sheet.

### What the cluster pins

| Image | Pinned by | Notes |
|---|---|---|
| `postgres:16` (postgres, backup) | digest ✅ | |
| `ghcr.io/jsundquist/sbom`, `sbom-worker` | tag only (`c79019e-4`) | bump bot writes a tag, not a digest |
| `ghcr.io/jsundquist/jsundquist.github.io` | tag only (`3c8e15a-47`) | same |
| `rclone/rclone:1.75.2` (backup) | version tag only | not digest-pinned |
| Helm charts (sealed-secrets 2.22.0, cert-manager) | exact chart version | not covered by Dependabot |

The `nginx` and `whoami` apps belong to the local practice cluster, not the droplet.

### Your strongest results

- **SBOM.** You built the dashboard and the upload action, and four of the five repos feed it. The weekly cluster upload watches what actually runs.
- **Repo hygiene.** SHA-pinned actions, least-privilege tokens, zizmor and actionlint, a 7-day Dependabot cooldown and gitleaks are close to uniform.
- **`sbom` repo.** It has the best container pipeline: scan-then-push of the identical image, digest-pinned base, `minimumReleaseAge`, narrow audit ignores with reasons, and a ruleset with no bypass.
- **Runtime hardening in the cluster.** Non-root pods, read-only rootfs, default-deny network policies, sealed secrets.

---

## Part 2: Per-repo to-do list

Items are ordered by value. Items marked **new** were found during this audit.

### sbom

- [ ] Sign both images with cosign keyless (job needs `id-token: write`; sign by digest, not by tag)
- [ ] Attest build provenance for both images (`actions/attest-build-provenance`, needs `attestations: write`; push by digest so the subject is exact)
- [ ] Generate an SBOM per image (Syft, which the action already uses) and attach it as a signed attestation
- [ ] Capture the pushed digest in `release.yml` and expose it as a job output; have `bump` write `tag@digest`
- [ ] **New:** triage the 2 open Dependabot alerts (both medium, dev-scope in `pnpm-lock.yaml`): `minimatch` fixed in 11.0.1, `esbuild` fixed in 0.25.0
- [ ] **New:** `release.yml` runs on every push to `main` and does not depend on the CI or security workflows. The ruleset forces PRs and checks first, so it is acceptable today; note the assumption, or add a `workflow_run` / reusable-workflow gate if the ruleset ever loosens
- [ ] **New:** the Dockerfile runs `apt-get install --only-upgrade perl-base` at build time. That makes the digest-pinned base non-reproducible. Remove it when the pinned base catches up, or bump the base digest instead
- [ ] **New:** `security.yml` rebuilds both images (`image-scan`) and `release.yml` builds them again. Fine now; fold into one reusable build when you extract the golden path
- [ ] GHCR retention for `sbom` and `sbom-worker` (already listed as unimplemented in `sbom.md`); never delete signatures and attestations that kept images point to
- [ ] Add `SECURITY.md` (CodeQL default setup is already on)
- [ ] Tag a release of the upload action (`v1`, plus a moving major tag if wanted) so consumers stop pinning commits
- [ ] Align the zizmor action version with the other repos (0.7.1 here, 0.8.0 elsewhere); Dependabot will do it if it hasn't already

### jsundquist.github.io (blog)

The biggest uplift is here: it ships a public image with fewer controls than `sbom`.

- [ ] Scan the image with Trivy before pushing (build with `load: true`, scan, push), same pattern as `sbom`
- [ ] Pin `nginxinc/nginx-unprivileged` by digest and add the Dependabot `docker` ecosystem so the digest stays current
- [ ] Add gitleaks (CI over full history, as in `sbom`)
- [ ] **New:** add the source SBOM upload (`sbom.yml` plus the `SBOM_DASHBOARD_TOKEN` secret). Today only the built image is tracked, through `k8s-gitops`. Dependencies in `pnpm-lock.yaml` are not
- [ ] **New:** add `pnpm audit` (production in CI, everything weekly). Today only `dependency-review` runs, and only on PR diffs, so a CVE published later is missed
- [ ] Add `minimumReleaseAge` to `pnpm-workspace.yaml` as in `sbom`
- [ ] Sign the image, attest provenance, attach an image SBOM; `bump` writes `tag@digest`
- [x] "Require actions pinned to a full SHA" is on
- [ ] Add `SECURITY.md` (CodeQL default setup is already on)
- [ ] Review the auto-merge: it is restricted to direct dev-dependency patch and minor updates, which is the tightest of the two repos that auto-merge. The workflow comment correctly notes that build tooling can alter the output. Keep the 7-day cooldown
- [ ] Add `org.opencontainers.image.source` label to the Dockerfile (as in `sbom`) so the GHCR package links to the repo

### k8s-gitops

This repo is where signing and provenance would be enforced, so it matters most for the "secure golden path" story.

- [ ] **New:** no branch protection or ruleset on `main`. The bump bots push directly with `GITOPS_TOKEN`, so the options are a ruleset that lets that token bypass, or have the bots open PRs and auto-merge them after checks. Pick one and write it down
- [x] `rclone/rclone` pinned by digest (this PR)
- [ ] Pin the blog, `sbom` and `sbom-worker` images by digest; the bump jobs must write `tag@digest` first, so this follows signing and attestation
- [ ] **New:** Dependabot cannot see kustomize `images:` `newTag` fields, and Helm chart versions in `argocd/` Applications (sealed-secrets, cert-manager) are not covered. Decide between Dependabot `docker` (plain `image:` lines only) and Renovate (has kustomize, Argo CD and Helm managers). Renovate fits this repo better, and should keep the same 7-day delay (`minimumReleaseAge`)
- [x] CI check that fails any rendered image without `@sha256:` (this PR). The three bot-managed images are allow-listed in `ci.yml`; remove each entry when its release workflow pins by digest
- [ ] Admission policy (Kyverno), starting in audit mode: require digest, require valid cosign signature from the `sbom` and blog workflows, require provenance, enforce pod hardening cluster-wide
- [ ] Verify signatures and provenance in CI before a bump lands (`gh attestation verify` or `cosign verify`)
- [x] "Require actions pinned to a full SHA" is on
- [ ] Extend network policies to `argocd`, `cert-manager`, `kube-system` (already deferred in `hardening.md`)
- [x] `SECURITY.md` added and private vulnerability reporting on (this PR); CodeQL default setup already covers the `actions` language
- [ ] Document the signing identity and the verify command next to the deploy docs

### knowledgebase

- [x] Dependabot cooldown added (PR #13, merged)
- [ ] **New:** `pip-audit` runs only on push and PR. Add a weekly schedule, as `transcript-search-mcp` does, so CVEs published later are caught
- [ ] Add actionlint and zizmor
- [ ] Add gitleaks in CI and as a pre-commit hook
- [ ] Add `dependency-review` on PRs
- [ ] **New:** the SBOM action is pinned to commit `b9a97735` with a `# main` comment. Dependabot cannot bump a commit that is not a release. Move to a tag once the action has one
- [x] "Require actions pinned to a full SHA" is on
- [ ] Add `SECURITY.md` (CodeQL default setup is already on)
- [ ] Decide whether the package is ever distributed. If so, add `uv build`, provenance and a release SBOM. If not, record "no published artifact" here and leave the artifact items as n/a
- [ ] Consider the one-PR Dependabot grouping (`patterns: ["*"]`): it bundles majors with patches. Group minor and patch only

### transcript-search-mcp

The most complete repo without an image.

- [ ] **New:** `dependency-review` runs but is not a required check (the required list is lint, build, test, audit, zizmor, secrets, workflow-lint). Add it
- [ ] **New:** Dependabot auto-merge covers every non-major update, including runtime dependencies. The blog limits this to dev dependencies. Decide deliberately; the 7-day cooldown and required checks are the safety net. The workflow is also a `pull_request_target` trigger with a zizmor exemption; the comment explains why it is safe (it never checks out PR code), so keep it that way
- [ ] SBOM action pinned to commit `38774aeb` with no release tag to bump from; same fix as knowledgebase
- [ ] **Enable CodeQL default setup** (Settings, Code security; it offers `actions` and `python`). This is the only repo of the five without it
- [ ] Add `SECURITY.md`
- [ ] Same publishing decision as knowledgebase
- [ ] Use its `security.yml` as the Python reference when building the golden path

### Cross-repo

- [ ] Same value for the SBOM action ref in all repos, and a tag to point at
- [ ] Switch "require actions pinned to a full SHA" on for `k8s-gitops`, `knowledgebase` and the blog
- [ ] Enable CodeQL default setup on `transcript-search-mcp` (the other four already have it)
- [ ] Rotate-and-expire check for tokens: `sbom`'s `release.yml` has a `token-expiry` job for `GITOPS_TOKEN`; the blog uses a separate `GITOPS_TOKEN` and has no equivalent
- [ ] `gh auth refresh -s read:packages`, then confirm GHCR package visibility and which repos can write each package
- [ ] Add `SECURITY.md` to all five (a shared one in a `jsundquist/.github` repo covers every repo that lacks its own) and turn on private vulnerability reporting (Settings, Code security)
- [ ] Keep these docs in step: `hardening.md`, `sbom.md` and this file

---

## Part 3: Checklist for any repo, now or later

Copy into the first PR of a new repo, or the first PR you open on an existing one.

### 0. Repo bootstrap (before any code)

- [ ] Private or public chosen on purpose; default branch `main`
- [ ] Secret scanning and push protection on; Dependabot alerts and security updates on
- [ ] Settings, Actions: default token read-only; Actions cannot approve PRs; **require actions pinned to a full-length SHA**
- [ ] Ruleset or branch protection on `main`: PR required, required status checks (list them), no force-push, no deletion, no bypass actors
- [ ] `SECURITY.md` plus private vulnerability reporting on; `CODEOWNERS` if more than one person ever commits
- [ ] SAST: CodeQL default setup, if the language is supported (SonarCloud is an optional addition for quality gates and tech-debt tracking; check with `gh api repos/OWNER/REPO/code-scanning/default-setup`; default setup creates no workflow file)
- [ ] Repo secrets added by name only; every long-lived token is fine-grained, one repo, and has an expiry check

### 1. Artifact hardening

- [ ] Build only in CI, from a clean checkout, on a hosted runner
- [ ] Top-level `permissions: {}` (or `contents: read`); grant per job
- [ ] Every action pinned by full SHA with a version comment; `persist-credentials: false` on checkout (document exceptions inline)
- [ ] No `${{ }}` expressions in `run:`; pass through `env:`
- [ ] actionlint and zizmor in CI, both required checks
- [ ] Container: multi-stage, base image pinned by digest, non-root numeric `USER`, `HEALTHCHECK` or probes, read-only rootfs and dropped capabilities in the manifest
- [ ] Scan the image that will be pushed (build with `load: true`, Trivy gate on HIGH and CRITICAL fixable, then push)
- [ ] Push by digest and record the digest as a job output
- [ ] Sign by digest with cosign keyless (`id-token: write`)
- [ ] Verify before deploy: in CI before the bump, and in the cluster through admission policy

### 2. Package and dependency management

- [ ] Lockfile committed and enforced (`--frozen-lockfile`, `uv sync --frozen`/`--locked`)
- [ ] Dependabot (or Renovate) for every ecosystem in the repo: language packages, `github-actions`, `docker`, `docker-compose`, Helm and kustomize if present
- [ ] 7-day cooldown on every entry; group minor and patch, leave majors as separate PRs
- [ ] `pnpm`: `allowBuilds` list, `minimumReleaseAge`; `uv`: lock exclusions reviewed; pip/npm install scripts off in audit jobs (`--ignore-scripts`)
- [ ] Vulnerability audit on every push and PR, plus a weekly schedule (CVEs appear without commits)
- [ ] `dependency-review-action` on PRs, and make it a **required** check
- [ ] Every audit ignore carries a reason, a scope and a removal condition
- [ ] gitleaks: pre-commit hook plus CI over full history
- [ ] Downloaded tools pinned by version and SHA256
- [ ] Auto-merge only for what you would be comfortable merging without reading: say which dependency types and update levels, and keep the required checks behind it
- [ ] New dependency in a PR: check maintainers, release age, install scripts, and whether a smaller option exists

### 3. Artifact repositories

- [ ] One registry per artifact type (GHCR for images); the package is linked to its repo (`org.opencontainers.image.source` label)
- [ ] Visibility chosen deliberately: public images need no pull secret
- [ ] Push with the short-lived `GITHUB_TOKEN`; deploy references **digest**, not a mutable tag
- [ ] Retention policy: delete old untagged versions, but keep what signatures and attestations reference
- [ ] Access review: who can push, pull and delete, and which workflows hold `packages: write`
- [ ] If you add a real repository manager (Artifactory, Cloudsmith, Harbor): builds pull dependencies only through it, direct public pulls blocked, allow/deny policy for packages and licences, promotion between staging and production repositories, audit log kept

### 4. SBOM

- [ ] Source SBOM uploaded to the dashboard on every push to `main` (`SBOM_DASHBOARD_TOKEN` set)
- [ ] Image SBOM generated for each built image and tied to its digest
- [ ] Image SBOM attached to the image as a signed attestation (`actions/attest`)
- [ ] Every deployed image appears in `cluster-sbom.yml`, including third-party ones (postgres, rclone)
- [ ] Findings triaged in the dashboard; accepted risks recorded with a reason and a review date (VEX when you are ready to publish it)
- [ ] Project names follow the dashboard convention

### 5. SLSA

- [ ] Build is scripted and runs in CI from version control (Build L1 needs provenance to exist, so this is not L1 until provenance is attached)
- [ ] Provenance generated by the platform and signed (`actions/attest-build-provenance`) → L2
- [ ] Build runs in a reusable workflow the calling repo cannot alter, or with `slsa-github-generator` → L3
- [ ] Provenance verified before deployment (`gh attestation verify oci://… --owner jsundquist`, later in admission policy)
- [ ] State the claimed level per repo in its README, and what is missing for the next level

### 6. Secure golden paths

- [ ] New repos start from a template that already contains the section 0 to 5 controls
- [ ] The secure steps live in reusable workflows or composite actions, so a fix lands everywhere once
- [ ] Controls are required, not optional: required status checks and ruleset, so a repo cannot skip them
- [ ] The cluster enforces what CI promises (Kyverno or Pod Security Admission), so skipping CI still fails at deploy
- [ ] Deployment only through git and Argo CD; no manual `kubectl apply`
- [ ] A short written path (this document or a Backstage template) that a new repo can follow in under an hour
- [ ] A scorecard you re-run: this audit's tables, or OpenSSF Scorecard on each repo

---

## Part 4: Suggested order

1. **Signing, provenance and image SBOM in `sbom`'s `release.yml`.** Prove it on one repo, and have `bump` write `tag@digest`.
2. **Blog parity with `sbom`:** Trivy, digest-pinned base, gitleaks, `pnpm audit`, the missing source-SBOM upload, then the same signing and provenance steps.
3. **Fast hygiene sweep across all five** (an hour or two):
   - `knowledgebase` cooldown, weekly audit, actionlint, zizmor, gitleaks and dependency-review.
   - Make `dependency-review` required in `transcript-search-mcp`.
   - Turn on SHA pinning in settings where it is off.
   - Add `SECURITY.md` and private vulnerability reporting everywhere, and enable CodeQL on `transcript-search-mcp`.
   - Close the 2 `sbom` Dependabot alerts.
4. **k8s-gitops protections:** a ruleset or PR flow for the bump bots, digest pins everywhere, and Renovate (or equivalent) for charts and kustomize.
5. **Kyverno in audit mode,** then enforcing once signing and provenance are live for both images.
6. **Extract the golden path** into reusable workflows and a template repo, and move all five repos onto it. Use `sbom`'s `release.yml` for images and `transcript-search-mcp`'s `security.yml` for Python.
7. **Optional:** stand up Harbor or use Cloudsmith's free tier, to get hands-on with an artifact repository manager.
