# Pod Security Standards Checker Reusable Workflow

Validates that Helm charts meet Kubernetes [Pod Security Standards (PSS)](https://kubernetes.io/docs/concepts/security/pod-security-standards/) using the [psa-checker](https://github.com/mozilla/psa-checker) tool.

## Overview

Checks rendered Helm chart manifests against a specified Pod Security Standard level (`privileged`, `baseline`, or `restricted`).

## Inputs

| Name        | Required | Type   | Default        | Description                                              |
| ----------- | -------- | ------ | -------------- | -------------------------------------------------------- |
| `pss_level` | false    | string | `"restricted"` | PSS level to check against: `privileged`, `baseline`, or `restricted` |

## Prerequisites

**IMPORTANT**: This workflow requires rendered chart artifacts from the `validate-k8s-manifests` workflow. Use `needs: validate_k8s_manifests` when calling this workflow.

## Usage

### Basic usage with default (restricted) level

```yaml
name: Validate Kubernetes Manifests

on:
  pull_request:
    paths:
      - '**/k8s/**'

jobs:
  validate:
    uses: mozilla/deploy-actions/.github/workflows/validate-k8s-manifests.yml@main

  check-pss:
    needs: validate
    uses: mozilla/deploy-actions/.github/workflows/psa-checker.yml@main
```

### Custom PSS level

```yaml
name: Validate Kubernetes Manifests

on:
  pull_request:
    paths:
      - '**/k8s/**'

jobs:
  validate:
    uses: mozilla/deploy-actions/.github/workflows/validate-k8s-manifests.yml@main

  check-pss:
    needs: validate
    uses: mozilla/deploy-actions/.github/workflows/psa-checker.yml@main
    with:
      pss_level: baseline
```

## Troubleshooting

If validation fails, review the workflow output to identify violations and consult the [Pod Security Standards documentation](https://kubernetes.io/docs/concepts/security/pod-security-standards/) for requirements.
