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

Example caller:

```yaml
name: "Test Coverage"

on:
  pull_request:
    branches: [ master, main ]
  workflow_dispatch:

jobs:
  coverage:
    uses: decaf-ts/reusable-actions/.github/workflows/jest-coverage.yaml@main
    secrets: inherit
```

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
