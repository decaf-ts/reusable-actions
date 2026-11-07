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

## Shared Workflows

### Build, test, and release

* `codeql-analysis.yml` — CodeQL analysis.
* `jest-coverage.yaml` — PR test coverage run + report.
* `nodejs-build-prod.yaml` — production build + full test suite.
* `pages.yaml` — coverage, docs build, GitHub Pages deploy.
* `publish-on-release.yaml` — OIDC trusted npm publish on GitHub Release.
* `publish-android.yml` — Google Play / F-Droid / GitHub Releases (Android).
* `publish-ios.yml` — App Store Connect / TestFlight / GitHub Releases (iOS).
* `publish-tv.yml` — Google Play / F-Droid / GitHub Releases (Android TV).
* `release-on-merge-pr.yml` — bump, build, test, and push the release commit/tag.
* `release-on-tag.yaml` — release gates (Trivy + Snyk) then GitHub Release.
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

**Pin every caller and every composite reference to a full 40-character commit
SHA.** A moving branch such as `@master` lets any change to this repository run
arbitrary code in every consumer with `secrets: inherit` — the largest publish-
trust risk in the chain. Renovate's `github-actions` manager with
`pinDigests: true` keeps the pins current.

Example caller:

```yaml
name: "Test Coverage"

on:
  pull_request:
    branches: [ master, main ]
  workflow_dispatch:

jobs:
  coverage:
    uses: decaf-ts/reusable-actions/.github/workflows/jest-coverage.yaml@<full-commit-sha> # vX.Y.Z
    secrets: inherit
```

Coverage enforcement is **not** configured by the action: `npm run coverage`
runs jest with each module's `workdocs/reports/jest.coverage.config.cjs`, whose
`coverageThreshold` is the single source of truth (jest exits non-zero when it
is not met). The PR coverage report (artiomtr) is report-only.

## Workflow reference

### `publish-on-release.yaml`

Publishes the package with npm **trusted publishing (OIDC)**. The workflow
requests `id-token: write`; npm exchanges that OIDC token for a short-lived
publish credential. No long-lived `NPM_TOKEN` is read or required.

| Input | Default | Description |
|-------|---------|-------------|
| `node_version` | `24` | Node.js version (must be >= 22.14 for trusted publishing). |
| `staged-publishing` | `false` | Run `npm stage publish` and leave the package staged for human approval instead of publishing directly. |

Secrets: none required for the publish path. `CONSECUTIVE_ACTION_TRIGGER`,
`GH_PAT`, and `GH_TOKEN` remain declared for backward compatibility and for the
release-creation chain, but are not used to authenticate the npm publish.

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

### `release-on-tag.yaml`

| Input | Default | Description |
|-------|---------|-------------|
| `run-security-scans` | `true` | Run the Trivy + Snyk release gates. |
| `node_version` | `24` | Node.js version. |
| `trivy-severity` | `HIGH,CRITICAL` | Trivy gate severity filter. |
| `trivy-ignore-unfixed` | `true` | Ignore vulnerabilities without a fix. |
| `snyk-severity-threshold` | `high` | Minimum Snyk severity that blocks the release. |
| `snyk-fail-on` | `upgradable` | `all`, `upgradable`, or `patchable`. |
| `strict-out-of-sync` | `true` | Require the lockfile to match package.json. |

Secrets: `GH_PAT`, `GH_TOKEN`, `CONSECUTIVE_ACTION_TRIGGER`, `SNYK_TOKEN`.

## Integration-test infrastructure contract (`prepare-it-tests`)

Every shared workflow that executes tests (`jest-coverage.yaml`,
`nodejs-build-prod.yaml`, `pages.yaml`, `release-on-merge-pr.yml`,
`publish-on-release.yaml`) runs:

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

## Invocation contract (one run per merge)

`release-on-merge-pr.yml` pushes an automatic release-bump commit to master
(`Github Action automatic release: vX.Y.Z`) together with the release tag.
That second push would re-trigger every push-triggered workflow, duplicating
runs. The shared workflows keep **exactly one executing invocation of each
workflow per PR merge** (not counting the PR checks):

* `pages.yaml` skips push events whose head commit message starts with
  `Github Action automatic release` (the run is created by the trigger but
  the job is skipped — zero execution).
* `trivy-scan.yml` and `pages.yaml` carry the same skip condition as
  defense-in-depth for repos that keep push triggers on them.
* Recommended caller pattern: deploy pages on the release tag push only
  (every merge releases), so each merge yields exactly one docs deployment.

The `Github Action automatic release` commit-message prefix is therefore a
contract between the shared workflows — do not rename it in
`release-on-merge-pr.yml` without updating the skip conditions.

## Security scanning model (release gates)

The snyk and trivy scans are **gates** inside `release-on-tag.yaml`, on the
tag-push event:

```text
PR merge -> release-on-merge-pr (bump + tag)
        -> tag push -> release-on-tag: [trivy gate] [snyk gate] -> GitHub Release
        -> release published -> publish-on-release (npm trusted publish)
```

* A failing gate **blocks the GitHub Release from being created**, which in
  turn blocks the npm publish — nothing reaches the registry.
* Gate failures dispatch `renovate-trigger` so security-remediation PRs are
  opened automatically.
* The snyk SARIF upload to code scanning is reporting-only and never blocks
  a release by itself.
* Snyk analyzes the npm manifest and lockfile directly; the gate does not run
  `npm ci`. `--strict-out-of-sync=true` fails the gate when the lockfile and
  `package.json` disagree instead of scanning a stale graph.
* Callers also run the standalone scans at their own cadence (the workspace
  caller runs Trivy report-only on every push/PR and Snyk report-only on every
  PR). Those runs are advisory; only the release gates block a publish.
* The standalone `trivy-scan.yml` workflow also provides the weekly
  dependency-update pass (`scan-type: dep`) that drives the renovate dep run.

## Secret fallbacks

Shared workflows resolve the GitHub PAT through a fallback chain
(`GH_PAT` -> `GH_TOKEN` -> `CONSECUTIVE_ACTION_TRIGGER`, where applicable) so
callers can `secrets: inherit` regardless of which PAT secret name the repo
has configured.

`NPM_TOKEN` is **not** used by the npm publish path: `publish-on-release.yaml`
publishes with OIDC trusted publishing (`id-token: write`). `NPM_TOKEN` still
applies to `docker-publish.yml`, where it is mounted only as the Docker build
secret `TOKEN` for private-registry installs.

## Reuse Rules

* Keep repository-specific event triggers in the caller workflow.
* Put shared implementation steps inside `reusable-actions/.github/workflows`
  and reusable step sequences inside `reusable-actions/actions`.
* Use `secrets: inherit` when the shared workflow needs access to existing secret names.
* Leave repository-specific workflows local when they are tied to unique runtime environments or services.
* Pin every reusable workflow, composite action, and third-party action
  reference to a full commit SHA (with the resolved version in a trailing
  comment). Renovate keeps the pins current.

## Validation Checklist

When updating a consumer or a shared workflow:

* Confirm the shared file still uses `workflow_call`.
* Confirm every `uses:` (including `decaf-ts/reusable-actions/actions/*`) is a
  full commit SHA.
* Confirm the caller still exposes the original trigger surface.
* Confirm any required secrets are available to the caller and inherited into the shared workflow.
* Confirm the OIDC publish path is intact: `id-token: write`, npm >= 11.5, and
  a provenance statement on the published package.
* Confirm repo-specific workflows that were intentionally excluded remain local.
* Confirm the shared workflow keeps the same execution behavior for the baseline jobs.
