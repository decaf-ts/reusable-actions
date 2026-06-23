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
