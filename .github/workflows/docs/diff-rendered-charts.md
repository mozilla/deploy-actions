# Render and Diff Helm Charts Reusable Workflow

Renders Helm charts from both the base and head refs of a pull request and posts a diff as a PR comment showing what changes will be deployed.

## Overview

- Detects changed Helm charts in pull requests
- Renders charts with all values files (supports multi-layer configurations)
- Posts a unified diff as a PR comment, minimizing the previous diff comment on each push
- Uploads `diff.log` as an artifact and exposes it to later jobs

## Inputs

| Name | Type | Default | Description |
|---|---|---|---|
| `automerge_test` | boolean | `false` | Run conftest against the structured diff and set a `conftest test` commit status (Dependabot automerge) |

## Outputs

| Name | Description |
|---|---|
| `changes_found` | `'true'` when at least one rendered chart differs between base and head |
| `diff_artifact` | Name of the artifact containing `diff.log` (`helm-diff-<pr number>`), empty when nothing changed |

Chain a [`claude-review`](./claude-review.md) job on these outputs to have the diff reviewed:

```yaml
  claude-review:
    needs: diff-charts
    if: needs.diff-charts.outputs.changes_found == 'true'
    uses: mozilla/deploy-actions/.github/workflows/claude-review.yml@main
    permissions:
      contents: read
      pull-requests: write
    with:
      helm_diff_artifact: ${{ needs.diff-charts.outputs.diff_artifact }}
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

The diff comment carries the hidden marker `<!-- helm-diff -->`; earlier diff comments (including
ones from before the marker existed) are minimized as outdated when a new one is posted.

## Usage

Call this workflow from your repository's pull request workflow:

```yaml
name: Review Helm Chart Changes

on:
  pull_request:
    paths:
      - '**/k8s/**'

jobs:
  diff-charts:
    uses: mozilla/deploy-actions/.github/workflows/diff-rendered-charts.yml@main
```

## Example Output

When changes are detected, a comment will be posted to the PR:

```diff
Changes found in Helm charts.

Changes found in chart: apps/my-service/k8s
--- shared/base-charts/apps/my-service/k8s/values-prod/my-service/templates/deployment.yaml
+++ shared/head-charts/apps/my-service/k8s/values-prod/my-service/templates/deployment.yaml
@@ -15,7 +15,7 @@
       containers:
       - name: my-service
-        image: my-service:v1.0.0
+        image: my-service:v1.1.0
```
