# Render Helm Charts and Validate Kubernetes Manifests Reusable Workflow

Renders Helm charts and validates the resulting Kubernetes manifests using [kubeconform](https://github.com/yannh/kubeconform). Posts validation failures as PR comments and uploads rendered manifests as artifacts for use by other workflows.

## Inputs

| Name     | Required | Type    | Default | Description                              |
| -------- | -------- | ------- | ------- | ---------------------------------------- |
| `strict` | false    | boolean | `false` | Run kubeconform with strict validation   |

### Strict Mode

When `strict: true`, kubeconform will reject resources with unknown fields and enforce stricter schema validation.

## Usage

### Basic validation

```yaml
name: Validate Kubernetes Manifests

on:
  pull_request:
    paths:
      - '**/k8s/**'

jobs:
  validate:
    uses: mozilla/deploy-actions/.github/workflows/validate-k8s-manifests.yml@main
```

### Strict validation

```yaml
name: Validate Kubernetes Manifests

on:
  pull_request:
    paths:
      - '**/k8s/**'

jobs:
  validate:
    uses: mozilla/deploy-actions/.github/workflows/validate-k8s-manifests.yml@main
    with:
      strict: true
```

### With pod security checking

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

## Example Output

When validation fails, a comment is posted to the PR:

```text
Kubernetes Manifest Validation: 2 resources found - Valid: 0, Invalid: 2, Errors: 0, Skipped: 0

apps/my-service/k8s/values-prod/my-service/templates/deployment.yaml - Deployment my-service failed validation: missing required field "selector" in io.k8s.api.apps.v1.Deployment
apps/my-service/k8s/values-prod/my-service/templates/service.yaml - Service my-service failed validation: Invalid value: "LoadBalancerr": spec.type
```

## Artifacts

Rendered manifests are uploaded as artifacts with the pattern `k8s-manifests-*` and can be consumed by other workflows like `psa-checker`.
