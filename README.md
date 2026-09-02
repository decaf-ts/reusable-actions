# reusable-actions

Workspace repository scaffold for shared GitHub Actions workflows.

## Layout

```text
.github/
  workflows/
```

## Shared Workflows

The baseline workflow set extracted from the workspace currently lives here:

* `codeql-analysis.yml`
* `jest-coverage.yaml`
* `nodejs-build-prod.yaml`
* `pages.yaml`
* `publish-on-release.yaml`
* `release-on-merge-pr.yml`
* `release-on-tag.yaml`
* `snyk-analysis.yaml`
* `trivy-scan.yml` — vulnerability / dependency-update scan (Trivy)
* `renovate.yml` — security remediation PRs (Renovate)

### `trivy-scan.yml`

A `workflow_call` reusable workflow that runs a Trivy filesystem scan. Callers
configure it through inputs:

| Input             | Default         | Description                                                                                                                                                                                                                                                                                                       |
|-------------------|-----------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `scan-type`       | `vuln`          | `vuln` = severity-filtered vulnerability scan that uploads `trivy-report.json`, counts findings, fails the job when `exit-code` is non-zero, and dispatches `renovate-trigger` on findings. `dep` = dependency-update pass (never fails) that dispatches `renovate-dep-trigger` to drive the weekly renovate run. |
| `severity`        | `HIGH,CRITICAL` | Comma-separated Trivy severity filter (vuln scan only).                                                                                                                                                                                                                                                           |
| `ignore-unfixed`  | `true`          | Ignore vulnerabilities without a known fix (vuln scan only).                                                                                                                                                                                                                                                      |
| `exit-code`       | `1`             | Trivy exit-code on found vulns (`0` = report only, `1` = fail the job). Ignored for `scan-type: dep`.                                                                                                                                                                                                             |
| `upload-artifact` | `true`          | Upload `trivy-report.json` as a workflow artifact.                                                                                                                                                                                                                                                                |

Secrets: `GH_PAT` (optional) — used to dispatch the renovate repository event. Pass with `secrets: inherit`.

### `renovate.yml`

A `workflow_call` reusable workflow that runs `renovatebot/github-action`. The
caller selects the remediation PR strategy through inputs:

| Input                   | Default           | Description                                                                                                                                                                                                                                                         |
|-------------------------|-------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `pr-strategy`           | `bump-dependents` | `overrides` = pin the offending library + its next safe version into `package.json` `overrides` (npm transitive remediation). `bump-dependents` = bump the dependency that brings in the offending library to its last safe release. `both` = apply both rule sets. |
| `clear-stale-overrides` | `false`           | Remove `package.json` `overrides` entries whose package no longer appears as vulnerable in the latest `trivy-report.json` (used by the weekly dep run).                                                                                                             |
| `renovate-config-file`  | `renovate.json`   | Base Renovate config in the caller repo. Strategy rules are merged into a generated `renovate-runtime.json` that Renovate actually consumes.                                                                                                                        |
| `target-branch`         | `master`          | Branch to check out and target as Renovate's base branch for remediation PRs.                                                                                                                                                                                        |
| `severity`              | `HIGH,CRITICAL`   | Comma-separated severity filter. Remediation is scoped (via `matchPackageNames`) to only the packages `trivy-report.json` flags at one of these severities; without a report there's nothing to scope by, so every security-category package is remediated as before this input existed.                        |

Secrets: `RENOVATE_TOKEN` (required for Renovate auth) and `GH_PAT` (optional).
Pass with `secrets: inherit`.

Renovate config knobs used:

* `extends: ["security:only-security-updates"]` (from the caller's base config) — only open security update PRs.
* `transitiveRemediation: true` — enables npm remediation of transitive vulnerabilities via the `overrides` field (Strategy A).
* `packageRules[].rangeStrategy: "replace"` (Strategy A) / `"bump"` (Strategy B).
* `packageRules[].automerge: true, automergeType: "pr"` — automerge security PRs once CI is green.

The stale-override cleanup cross-references `trivy-report.json` (produced by
`trivy-scan.yml`) and removes override entries whose package is no longer
vulnerable, keeping the `overrides` block only as wide as the active
vulnerability surface.

## Consumer Pattern

Repository-level workflows now act as thin callers that trigger the shared reusable workflow definitions from this folder once the repository is published as `decaf-ts/reusable-actions`.

The shared repo's branch is `master` — pin callers to `@master`.

Example caller:

```yaml
name: "Test Coverage"

on:
  pull_request:
    branches: [ master, main ]
  workflow_dispatch:

jobs:
  coverage:
    uses: decaf-ts/reusable-actions/.github/workflows/jest-coverage.yaml@master
    secrets: inherit
```

## Integration-test infrastructure contract (`prepare-it-tests`)

Every shared workflow that executes tests (`jest-coverage.yaml`, `nodejs-build-prod.yaml`, `pages.yaml`, `release-on-merge-pr.yml`, `publish-on-release.yaml`) runs:

```bash
npm run prepare-it-tests --if-present
```

before any test/coverage step. Repositories whose integration/e2e tests need
backing infrastructure (databases, containers, networks, ...) must define a
`prepare-it-tests` npm script that boots it. Repositories with no infra needs
may either omit the script (`--if-present` makes it optional) or define a
no-op for explicitness, e.g. `echo "prepare-it-tests: no infra to boot"`.

## Invocation contract (one run per merge)

`release-on-merge-pr.yml` pushes an automatic release-bump commit to master
(`Github Action automatic release: vX.Y.Z`) together with the release tag.
That second push would re-trigger every push-triggered workflow, duplicating
runs. To keep **exactly one executing invocation of each workflow per PR
merge** (not counting the PR checks):

* `trivy-scan.yml` and `pages.yaml` skip push events whose head commit
  message starts with `Github Action automatic release` (the run is created
  by the trigger but the job is skipped — zero execution).
* `snyk-analysis.yml` and `codeql-analysis.yml` run on the tag push (their
  push triggers are tag-only) and genuinely execute there.

The `Github Action automatic release` commit-message prefix is therefore a
contract between the shared workflows — do not rename it in
`release-on-merge-pr.yml` without updating the skip conditions.

## Secret fallbacks

Shared workflows resolve the GitHub PAT through a fallback chain
(`GH_PAT` -> `GH_TOKEN` -> `CONSECUTIVE_ACTION_TRIGGER`, where applicable) so
callers can `secrets: inherit` regardless of which PAT secret name the repo
has configured. `NPM_TOKEN` is always read directly (and mirrored into
`NODE_AUTH_TOKEN` for the `.npmrc` generated by `setup-node`'s
`registry-url`).

## Reuse Rules

* Keep repository-specific event triggers in the caller workflow.
* Put shared implementation steps inside `reusable-actions/.github/workflows`.
* Use `secrets: inherit` when the shared workflow needs access to existing secret names.
* Leave repository-specific workflows local when they are tied to unique runtime environments or services.
* Pin the reusable workflow reference to a stable branch or tag when the shared repo is published outside this workspace.

## Validation Checklist

When updating a consumer or a shared workflow:

* Confirm the shared file still uses `workflow_call`.
* Confirm the caller still exposes the original trigger surface.
* Confirm any required secrets are available to the caller and inherited into the shared workflow.
* Confirm repo-specific workflows that were intentionally excluded remain local.
* Confirm the shared workflow keeps the same execution behavior for the baseline jobs.
