# Render and Diff Helm Charts Reusable Workflow

Renders Helm charts from both the base and head refs of a pull request and posts a diff as a PR comment showing what changes will be deployed.

## Overview

- Detects changed Helm charts in pull requests
- Renders charts with all values files (supports multi-layer configurations)
- Posts a unified diff as a PR comment

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
