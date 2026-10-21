# Render Helm Charts and Validate Kubernetes Manifests Reusable Workflow

Renders Helm charts and validates the resulting Kubernetes manifests using [kubeconform](https://github.com/yannh/kubeconform). Posts validation failures as PR comments and uploads rendered manifests as artifacts for use by other workflows.

## Inputs

| Name     | Required | Type    | Default | Description                              |
| -------- | -------- | ------- | ------- | ---------------------------------------- |
| `strict` | false    | boolean | `false` | Run kubeconform with strict validation   |

### Strict Mode

When `strict: true`, kubeconform will reject resources with unknown fields and enforce stricter schema validation.

## How Charts Are Found and Rendered

A chart is any `<dir>/k8s/<chart>/` directory containing a `Chart.yaml` with at least one changed file (templates, values, `Chart.lock`, `.helmignore`, files loaded with `.Files.Get`, etc.). Changed paths under `k8s/` without a `Chart.yaml`, such as values-only directories for upstream charts, raw manifests, or deleted charts, are skipped with a notice.

Every top-level YAML file in the chart directory, other than `Chart.yaml`, `values.yaml` and dotfiles, is a values file for one deployment. The chart is rendered once per values file, layered on top of `values.yaml` and on any values file whose name is a prefix of its own:

- `values-prod.yaml` renders with `values.yaml` + `values-prod.yaml`
- `values-prod-us-west1.yaml` renders with `values.yaml` + `values-prod.yaml` + `values-prod-us-west1.yaml`
- A chart with no other values files is rendered with `values.yaml` alone

Each render is validated with kubeconform. Any error from `helm dependency update` or `helm template`, or any invalid manifest, fails the workflow.

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

When validation fails, kubeconform's output is posted to the PR as a comment, with a collapsible section for each failing render. If it is too long for a single comment, it is written to the workflow run summary instead, and the comment links to it.

````markdown
Kubernetes manifest validation failed:

<details><summary>apps/k8s/my-service (values-prod)</summary>

```text
my-service/templates/deployment.yaml - Deployment my-service is invalid: problem validating schema. Check JSON formatting: jsonschema validation failed with 'https://raw.githubusercontent.com/yannh/kubernetes-json-schema/master/master-standalone-strict/deployment-apps-v1.json#' - at '/spec/template/spec/containers/0/securityContext': additional properties 'fsGroupChangePolicy' not allowed
Summary: 12 resources found in 8 files - Valid: 11, Invalid: 1, Errors: 0, Skipped: 0
```

</details>
````

## Artifacts

Rendered manifests are uploaded as artifacts with the pattern `k8s-manifests-*` and can be consumed by other workflows like `psa-checker`.
