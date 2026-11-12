# reusable-actions

Workspace repository scaffold for shared GitHub Actions workflows and composite
actions.

## Layout

```text
actions/
  node-setup/
  trivy-gate/
  snyk-gate/
  renovate-dispatch/
  capacitor-android-build/
.github/
  workflows/
```

Why a separate `actions/` directory: GitHub offers exactly two reuse
primitives. Reusable **workflows** (`on: workflow_call`) must live under
`.github/workflows/` and are scheduled as their own runs. Reusable **step
sequences** are composite actions — a directory containing an `action.yml` —
which GitHub expects under `actions/` by convention. `node-setup`,
`trivy-gate`, `snyk-gate`, `renovate-dispatch`, and
`capacitor-android-build` are composite actions because they are *steps*
shared by several workflows; they are inlined into the calling job and add
no runner start, scheduling, or billing of their own. They exist to remove
copy-pasted `checkout`/`setup-node`/cache/scan blocks (and the cache-key
drift that comes with them), not to add CI runs.

## Shared Workflows

### Build, test, and release

The pipeline is deliberately four events, one build/test each:

* `jest-coverage.yaml` — the **single PR check**: `npm ci` → `build:prod` →
  `coverage` (one build, one full test run) → coverage report. Display name
  `CI`.
* `release-on-merge-pr.yml` — on a merged PR: bump the version, commit, and
  push the tag (no build, no test — the PR already ran `CI`). The tag is
  pushed with the **PAT**, so the tag push fires the tag caller.
* `release-on-tag.yaml` — on a tag push: the Trivy + Snyk **release gates**,
  then the **GitHub Release** (created with the PAT so the `release` event
  fires). No build, no publish: it only gates and creates the Release.
* `publish-on-release.yaml` — triggered by the `release: published` event via
  **each repository's own `publish-on-release.yaml` caller**. It runs the
  single release build/test (`npm ci` → `build:prod` → `test:all`), generates
  and deploys the docs, and performs the **actual `npm publish`** with OIDC
  trusted publishing. The publishing repository's caller **must keep the
  filename `publish-on-release.yaml`** — see "npm trusted publishing" below.
* `codeql-analysis.yml` — CodeQL analysis.
* `publish-android.yml` — Google Play / F-Droid / GitHub Releases (Android).
* `publish-ios.yml` — App Store Connect / TestFlight / GitHub Releases (iOS).
* `publish-tv.yml` — Google Play / F-Droid / GitHub Releases (Android TV).
* `build-android.yml` — Capacitor Android APK/AAB build.
* `build-ios.yml` — Capacitor iOS `.app` build.
* `build-tv.yml` — Capacitor Android TV / leanback build.

### Security and automation

* `snyk-analysis.yaml` — Snyk scan (report-only) with SARIF upload.
* `trivy-scan.yml` — Trivy vulnerability / dependency-update scan.
* `renovate.yml` — security remediation PRs (Renovate).
* `auto-merge-renovate.yml` — waits for checks, approves, and merges bot PRs.
* `bug-in-progress.yml` — Jira automation: create the bug branch.
* `bug-pull-request.yml` — Jira automation: open the bug PR and request reviewers.
* `docker-publish.yml` — Docker image build/publish (optional build, test, Trivy gate).

## Composite Actions

| Action | Purpose |
|--------|---------|
| `actions/node-setup` | Checkout → `setup-node` with the built-in npm cache → optional `npm ci`. Replaces the hand-rolled `actions/cache` blocks. |
| `actions/trivy-gate` | One Trivy invocation shape with a cached vulnerability DB, optional report output, and configurable exit code. |
| `actions/snyk-gate` | One Snyk invocation shape with a `fail` input (gate vs report-only) and SARIF output. |
| `actions/renovate-dispatch` | One `repository_dispatch` helper with a configurable event type. |
| `actions/capacitor-android-build` | Shared Capacitor web build + Android APK/AAB assembly, optional TV manifest patch and release signing. |

## Consumer Pattern

Repository-level workflows act as thin callers of the shared reusable workflow
definitions in this repository.

Reference shared workflows and composite actions by their **version tag**, not
by a commit hash: this repository's own workflows/composites use `@master`, and
third-party actions use their latest major tag (`@v7`, `@v6`, ...). Renovate's
`github-actions` manager keeps those references current. (If a consumer needs
immutable references, pin to a release tag such as `@v1` rather than a SHA.)

The repository keeps four thin callers — `jest-coverage.yaml` (PR),
`release-on-merge-pr.yml` (merge), `release-on-tag.yaml` (tag) and
`publish-on-release.yaml` (release). Example:

```yaml
name: "CI"

on:
  pull_request:
    branches: [ master, main ]
  workflow_dispatch:

jobs:
  build-test:
    uses: decaf-ts/reusable-actions/.github/workflows/jest-coverage.yaml@master
    secrets: inherit
```

Coverage enforcement is **not** configured by the action: `npm run coverage`
runs jest with each module's `workdocs/reports/jest.coverage.config.cjs`, whose
`coverageThreshold` is the single source of truth (jest exits non-zero when it
is not met). The PR coverage report (artiomtr) is report-only.

## Workflow reference

### `release-on-tag.yaml` (gates + GitHub Release)

Runs the Trivy + Snyk **release gates** on a tag push, then creates the
**GitHub Release**. It does **not** build, test, publish, or deploy docs: the
Release it creates fires the `release: published` event, which triggers each
repository's own `publish-on-release.yaml` caller.

A failing gate fails the job before the Release step, so no Release is created
and `publish-on-release.yaml` never runs — the gate failure therefore blocks
the npm publish.

```text
tag push
  -> gates:  [trivy] [snyk]        (scan only, no build)
  -> Release [PAT-authored]        (fires release: published)
```

The Release **must** be created with a PAT (`GH_PAT`/`GH_TOKEN`/
`CONSECUTIVE_ACTION_TRIGGER`), not the workflow token: Releases created with
`GITHUB_TOKEN` do not fire the `release` event and would never trigger
`publish-on-release.yaml`.

| Input | Default | Description |
|-------|---------|-------------|
| `node_version` | `24` | Node.js version used by the gate tooling. |
| `run-security-scans` | `true` | Run the Trivy + Snyk release gates. |
| `trivy-severity` | `HIGH,CRITICAL` | Trivy gate severity filter. |
| `trivy-ignore-unfixed` | `true` | Ignore vulnerabilities without a fix. |
| `snyk-severity-threshold` | `high` | Minimum Snyk severity that blocks the release. |
| `snyk-fail-on` | `upgradable` | `all`, `upgradable`, or `patchable`. |
| `strict-out-of-sync` | `true` | Require the lockfile to match package.json. |

Secrets: `GH_PAT`, `GH_TOKEN`, `CONSECUTIVE_ACTION_TRIGGER` (Release creation
and renovate dispatch), `SNYK_TOKEN` (Snyk gate). None of the PATs authenticate
the npm publish: OIDC does.

Caller permissions: `contents: write` (create the Release) and
`security-events: write` (SARIF upload).

### `publish-on-release.yaml` (npm publish — DO NOT RENAME, DO NOT MOVE PUBLISH)

The workflow that runs the **actual `npm publish`**. It owns the single
release build/test (`npm ci` → `build:prod` → `test:all`), generates and
deploys the docs, and publishes with OIDC trusted publishing
(`npm publish --provenance`).

It is invoked by **each package repository's own
`.github/workflows/publish-on-release.yaml` caller**, triggered by the
`release: published` event:

```text
release: published
  -> <repo>/.github/workflows/publish-on-release.yaml   (caller; entry workflow)
       -> decaf-ts/reusable-actions/.github/workflows/publish-on-release.yaml
            [npm ci] [build:prod] [test:all]      (one build/test)
            [docs] -> upload pages artifact
            [npm publish --provenance]            (OIDC trusted publishing)
       -> pages job: deploy docs artifact          (no rebuild)
```

**npm trusted publishing validates the CALLING workflow's filename.** The npm
docs are explicit: with `workflow_call`, "validation checks the calling
workflow's name instead of the workflow that actually contains the publish
command". Every decaf-ts package is configured on npm as
`repository=<package repo>`, `workflow=publish-on-release.yaml`. That is why
each repository keeps its own `publish-on-release.yaml` caller and why the
publish must not be moved into `release-on-tag.yaml` or
`release-on-merge-pr.yml`: that would change the calling workflow name and
break the trusted publisher for **every** package, requiring npm
reconfiguration.

`id-token: write` must be granted by **both** the calling workflow and this
reusable workflow.

| Input | Default | Description |
|-------|---------|-------------|
| `node_version` | `24` | Node.js version (must be >= 22.14 for trusted publishing). |
| `ref` | `""` | Tag/ref to build and publish. Empty = the triggering ref (the release tag). |
| `staged-publishing` | `false` | Run `npm stage publish` instead of a direct publish. |
| `deploy-docs` | `true` | Build docs and deploy them to GitHub Pages. |

Permissions required from the caller of `publish-on-release.yaml`:
`contents: read`, `packages: read`, `id-token: write` (npm OIDC),
`pages: write` (docs deploy).

### `build-android.yml` / `build-tv.yml` / `build-ios.yml`

| Input | Default | Description |
|-------|---------|-------------|
| `build_type` | `debug` | Android build type (`debug` or `release`). |
| `signing_enabled` | `false` | Configure release signing from the keystore secrets (Android). |
| `patch_tv_manifest` | `true` | Patch the manifest for leanback/D-pad devices (TV only). |
| `configuration` | `Debug` | Xcode configuration (iOS only). |
| `code_signing_allowed` | `false` | Allow code signing during `xcodebuild` (iOS only). |
| `node_version` | `24` | Node.js used to build the web shell. |
| `java_version` | `21` | Java used by the Gradle build (Android/TV). |
| `upload_artifacts` | `true` | Upload the build artifacts. |
| `artifact_name` | platform-specific | Uploaded artifact name. |
| `web_build_command` | `npm run web:build` | Command that builds the Capacitor web shell. |
| `keystore_file` | `android/upload.keystore` | Where the decoded keystore is written (Android). |

Android secrets: `ANDROID_UPLOAD_KEYSTORE`,
`ANDROID_UPLOAD_KEYSTORE_PASSWORD`, `ANDROID_UPLOAD_KEY_ALIAS`,
`ANDROID_UPLOAD_KEY_PASSWORD`. The TV workflow accepts them for caller
compatibility but builds unsigned.

### `publish-android.yml` / `publish-tv.yml` / `publish-ios.yml`

| Input | Default | Description |
|-------|---------|-------------|
| `store_targets` | platform-specific | Comma-separated stores: `google_play`, `fdroid`, `github_releases`, `app_store_connect`. |
| `app_id` | `com.flipbored.reader` | Android applicationId / iOS bundle identifier. |
| `app_name` | `flipbored` | Human-readable name used in release names/bodies and F-Droid metadata. |
| `version_name` / `version_code` | `0.1.0` / `1` | Android version metadata. |
| `artifact_name` | platform-specific | Build artifact to download and publish. |
| `release_tag` | derived | GitHub Releases tag; defaults to `v<version_name>[-ios|-tv]`. |
| `fdroid_repo` / `fdroid_commit` | flipbored repo / `master` | F-Droid source repository and build ref (Android/TV). |
| `fdroid_categories` / `fdroid_summary` / `fdroid_index_repo` | metadata defaults | F-Droid metadata for the Android path. |
| `scheme` / `export_method` | `App` / `app-store-connect` | iOS archive/export settings. |

Secrets: `GOOGLE_PLAY_SERVICE_ACCOUNT_JSON`, the `ANDROID_UPLOAD_*` keystore
set, `FDROID_FDROIDDATA_REPO`, `FDROID_FDROIDDATA_TOKEN`,
`APPLE_DISTRIBUTION_CERTIFICATE_B64`,
`APPLE_DISTRIBUTION_CERTIFICATE_PASSWORD`, `APPLE_TEAM_ID`, `APPLE_API_KEY`,
`APPLE_API_KEY_ID`, `APPLE_API_ISSUER`. Each store is independently
credential-gated: a missing secret skips only that store.

### `docker-publish.yml`

| Input | Default | Description |
|-------|---------|-------------|
| `image` | required | Image path under each registry. |
| `registries` | `ghcr.io` | Comma-separated registries. |
| `registry-user` | `decaf-ts-admin` | Registry login user. |
| `dockerfile` / `context` | `Dockerfile` / `.` | Build inputs. |
| `tag-prefix` / `floating-tag` | `""` / `latest` | Tag shape. |
| `version-tag` | package.json | Explicit version tag. |
| `push` | `true` | Push the built image. |
| `platforms` | `linux/amd64` | Buildx platforms. |
| `pre-build-command` | `""` | Command run before the image build. |
| `no-cache` | `false` | Disable layer cache reuse. |
| `run-build-test` | `false` | Run `npm ci`, `build:prod`, `prepare-it-tests`, `test:all`. |
| `run-trivy-scan` | `false` | Run the Trivy gate before building. |
| `scan-severity` | `HIGH,CRITICAL` | Trivy severity filter. |
| `node_version` | `24` | Node version for build/test. |

Secrets: `GH_PAT`, `GH_TOKEN`, `NPM_TOKEN` (mounted only as the Docker build
secret `TOKEN`), `REGISTRY_TOKEN`.

### `auto-merge-renovate.yml`

| Input | Default | Description |
|-------|---------|-------------|
| `allowed-actors` | `renovate[bot],dependabot[bot]` | Actors permitted to trigger an automatic merge; the job fails fast for anything else. |

Secret: `GH_PAT` (approve/merge/check-read).

### `bug-in-progress.yml` / `bug-pull-request.yml`

| Input | Description |
|-------|-------------|
| `branch` | Bug branch to create, or the branch to open the PR from. |

Secret: `JIRA_AUTOMATION` (reviewer lookup) for `bug-pull-request.yml`.

### `trivy-scan.yml`

A `workflow_call` reusable workflow that runs a Trivy filesystem scan. Callers
configure it through inputs:

| Input | Default | Description |
|-------|---------|-------------|
| `scan-type` | `vuln` | `vuln` = severity-filtered vulnerability scan that uploads `trivy-report.json`, counts findings, fails the job when `exit-code` is non-zero, and dispatches `renovate-trigger` on findings. `dep` = dependency-update pass (never fails) that dispatches `renovate-dep-trigger` to drive the weekly renovate run. |
| `severity` | `HIGH,CRITICAL` | Comma-separated Trivy severity filter (vuln scan only). |
| `ignore-unfixed` | `true` | Ignore vulnerabilities without a known fix (vuln scan only). |
| `exit-code` | `1` | Trivy exit-code on found vulns (`0` = report only, `1` = fail the job). Ignored for `scan-type: dep`. |
| `upload-artifact` | `true` | Upload `trivy-report.json` as a workflow artifact. |
| `target-branch` | `master` | Branch to check out and scan. |

Secrets: `GH_PAT` (optional) — used to dispatch the renovate repository event.
Pass with `secrets: inherit`.

### `renovate.yml`

| Input | Default | Description |
|-------|---------|-------------|
| `pr-strategy` | `bump-dependents` | `overrides` = pin the offending library + its next safe version into `package.json` `overrides`. `bump-dependents` = bump the dependency that brings in the offending library. `both` = apply both rule sets. |
| `clear-stale-overrides` | `false` | Remove `package.json` `overrides` entries whose package no longer appears as vulnerable in the latest `trivy-report.json`. |
| `renovate-config-file` | `renovate.json` | Base Renovate config merged into `renovate-runtime.json`. |
| `target-branch` | `master` | Branch to check out and target as Renovate's base branch. |
| `severity` | `HIGH,CRITICAL` | Severity filter used to scope `matchPackageNames`. |
| `node_version` | `24` | Node.js version. |

Secrets: `RENOVATE_TOKEN` (Renovate auth) and `GH_PAT` (optional).

The stale-override cleanup cross-references `trivy-report.json` (produced by
`trivy-scan.yml`) and removes override entries whose package is no longer
vulnerable.

## Integration-test infrastructure contract (`prepare-it-tests`)

Every shared workflow that executes tests (`jest-coverage.yaml`,
`publish-on-release.yaml`, and `docker-publish.yml` when `run-build-test` is
enabled) runs:

```bash
npm run prepare-it-tests --if-present
```

before any test/coverage step. Repositories whose integration/e2e tests need
backing infrastructure (databases, containers, networks, ...) must define a
`prepare-it-tests` npm script that boots it. Repositories with no infra needs
may either omit the script (`--if-present` makes it optional) or define a
no-op for explicitness, e.g. `echo "prepare-it-tests: no infra to boot"`.

The shared workflows no longer run a `docker login ghcr.io` step. If a
`prepare-it-tests` script pulls private base images, it must authenticate inside
the script itself (for example `docker login ghcr.io` using an inherited token)
rather than relying on a login step in the workflow.

## Invocation contract (one build/test per event)

The pipeline is four events, with the build and test run exactly once per
event:

| Event | Workflow | Build/test |
|-------|----------|------------|
| Pull request | `jest-coverage.yaml` | one `npm ci` + `build:prod` + `coverage` (which runs the full suite) |
| PR merged | `release-on-merge-pr.yml` | **none** — bump version + push tag only |
| Tag pushed | `release-on-tag.yaml` | **none** — Trivy + Snyk gates, then create the GitHub Release |
| Release published | `publish-on-release.yaml` (the repo's own caller) | one `npm ci` + `build:prod` + `test:all`; docs generated + deployed from that same build; `npm publish` in the same job |

`release-on-merge-pr.yml` pushes an automatic release-bump commit to master
(`Github Action automatic release: vX.Y.Z`) and the tag, both with the **PAT**,
so the tag push starts `release-on-tag.yaml`. `release-on-tag.yaml` then
creates the Release with the PAT, so the `release` event starts the repo's own
`publish-on-release.yaml` caller. Using the PAT (not `github.token`) at both
steps is required: `GITHUB_TOKEN`-authored pushes/releases do not trigger
workflows. Manual tags pushed with a PAT follow the same tag → Release →
publish chain.

The bump commit's push can still re-trigger push-triggered workflows, so
`trivy-scan.yml` skips push events whose head commit message starts with
`Github Action automatic release` (the run is created but the job is skipped —
zero execution).

The `Github Action automatic release` commit-message prefix is therefore a
contract between the shared workflows — do not rename it in
`release-on-merge-pr.yml` without updating the skip condition.

## Caching model (and why it never conflicts)

Reuse is deliberately limited to inputs that can be safely shared; build
outputs are never restored across commits, so a source-only PR can never test
a stale build.

| Cache | Key | Reused by |
|-------|-----|-----------|
| npm dependencies | `setup-node` `cache: npm`, keyed by the lockfile | every Node job (`node-setup` is the single install path) |
| Trivy DB | `~/.cache/trivy`, OS + suffix, `run_id` key with restore-keys | every trivy gate/scan |
| Gradle | `setup-java` `cache: gradle`, `android/**/*.gradle*` | Android/TV builds |
| Docker layers | buildx `type=gha,mode=max` | image builds |

Docs are passed from the `publish-on-release.yaml` `publish` job to its
`pages` job as a workflow artifact (`docs`), not rebuilt. npm publish runs in
the same job that built and tested, so there is no reinstall either. There is
no separate `pages.yaml`: docs are built once, on the release event.

## Security scanning model (release gates)

The Snyk and Trivy scans are **gates** inside `release-on-tag.yaml`:

```text
PR merge -> release-on-merge-pr   (bump + push tag)
tag push -> release-on-tag:
             [trivy gate] [snyk gate]            (scan only)
             -> [GitHub Release, PAT-authored]   (only if both gates pass)
release  -> publish-on-release.yaml:
             [build + test once]
             [docs build + deploy]
             [npm publish --provenance]
```

* A failing gate **blocks the GitHub Release**, so `publish-on-release.yaml`
  never runs: nothing reaches the registry and no docs deploy happens.
* Gate failures dispatch `renovate-trigger` so security-remediation PRs are
  opened automatically.
* The snyk SARIF upload to code scanning is reporting-only and never blocks
  a release by itself.
* Snyk analyzes the npm manifest and lockfile directly; the gate does not run
  `npm ci`. `--strict-out-of-sync=true` fails the gate when the lockfile and
  `package.json` disagree instead of scanning a stale graph.
* Callers also keep their original standalone scan triggers (the workspace
  caller runs Trivy report-only on every push/PR and Snyk report-only on every
  PR and tag push). Those runs are advisory; only the release gates block a
  publish.
* The standalone `trivy-scan.yml` workflow also provides the weekly
  dependency-update pass (`scan-type: dep`) that drives the renovate dep run.

## Secret fallbacks

Shared workflows resolve the GitHub PAT through a fallback chain
(`GH_PAT` -> `GH_TOKEN` -> `CONSECUTIVE_ACTION_TRIGGER`, where applicable) so
callers can `secrets: inherit` regardless of which PAT secret name the repo
has configured.

`NPM_TOKEN` is **not** used by the npm publish path:
`publish-on-release.yaml` publishes with OIDC trusted publishing
(`id-token: write`). `NPM_TOKEN` still applies to `docker-publish.yml`, where it
is mounted only as the Docker build secret `TOKEN` for private-registry
installs.

## Reuse Rules

* Keep repository-specific event triggers in the caller workflow.
* Put shared implementation steps inside `reusable-actions/.github/workflows`
  and reusable step sequences inside `reusable-actions/actions`.
* Use `secrets: inherit` when the shared workflow needs access to existing secret names.
* Leave repository-specific workflows local when they are tied to unique runtime environments or services.
* Reference reusable workflows and composite actions from this repository
  with a version tag (`@master` for the shared workflows/composites), and
  third-party actions with their latest major tag (`@vN`). Renovate keeps
  those references current.

## Validation Checklist

When updating a consumer or a shared workflow:

* Confirm the shared file still uses `workflow_call`.
* Confirm every `uses:` (including `decaf-ts/reusable-actions/actions/*`) uses
  a version reference (`@master` for shared definitions, latest `@vN` for
  third-party actions) rather than a commit hash.
* Confirm the caller still exposes the original trigger surface.
* Confirm any required secrets are available to the caller and inherited into the shared workflow.
* Confirm the OIDC publish path is intact: `id-token: write` on **both** the
  caller and `publish-on-release.yaml`, npm >= 11.5, and a provenance statement
  on the published package.
* Confirm the event still builds/tests only once: PR → `jest-coverage.yaml`,
  merged PR → bump/tag only, tag → gates + Release only, release →
  `publish-on-release.yaml` (one build/test with docs). Do not reintroduce a
  second `npm ci`/`build:prod`/test run.
* Confirm the tag **and** the GitHub Release are created with the **PAT**, not
  `github.token`, or the `release` event never fires and nothing publishes.
* Confirm the consumer still has its own `publish-on-release.yaml` caller on
  `release: published`, named exactly `publish-on-release.yaml`. npm trusted
  publishing validates the **calling** workflow filename, so this file must
  exist and must not be renamed. Do not move `npm publish` into
  `release-on-tag.yaml` / `release-on-merge-pr.yml`: that changes the calling
  workflow name and breaks every package's trusted publisher.
* Confirm the actual `npm publish` step lives only in
  `publish-on-release.yaml`.
* Confirm docs are passed to the `pages` job as an artifact, not rebuilt.
* Confirm the consumer no longer keeps `nodejs-build-prod.yaml` or `pages.yaml`:
  their work is folded into `jest-coverage.yaml` and `publish-on-release.yaml`.
* Confirm repo-specific workflows that were intentionally excluded remain local.
* Confirm the shared workflow keeps the same execution behavior for the baseline jobs.
