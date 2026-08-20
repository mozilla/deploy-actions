# Render and Diff Helm Charts Reusable Workflow

Renders Helm charts from both the base and head refs of a pull request and posts a semantic diff as a PR comment showing what changes will be deployed.

## Overview

- Detects changed Helm charts in pull requests
- Renders charts with all values files (supports multi-layer configurations)
- Compares the rendered manifests with [dyff](https://github.com/homeport/dyff), which matches resources on `apiVersion`, `kind`, and `metadata.name`
- Posts a per-resource summary with the full diff in a collapsible block
- Folds away diff comments from previous pushes so only the current one is expanded

## Why a semantic diff

Rendered manifests used to be compared with `diff -ruN` over the two output directories, which makes a template's **filename** the de facto identity of a resource. Moving a template to a new path was therefore reported as an entire resource being deleted and an unrelated one being added, even when nothing about the resource actually changed.

That is the normal shape of a MozCloud chart migration, where templates move from `chart/templates/*.yaml` to `chart/charts/mozcloud/templates/**.yaml`. Because most or all templates move at once, the old output was close to useless for review.

dyff keys documents on `apiVersion`/`kind`/`metadata.name`, so resources are matched across path moves and the diff shows only the fields that actually changed. It also ignores list ordering and whitespace-only differences.

For the same behavior locally, `render-diff --semantic` (from [mozilla/mozcloud](https://github.com/mozilla/mozcloud)) wraps the same dyff comparison.

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
    permissions:
      contents: read
      pull-requests: write
      statuses: write
    uses: mozilla/deploy-actions/.github/workflows/diff-rendered-charts.yml@main
```

`pull-requests: write` is required both to post the comment and to minimize superseded ones.

## Inputs

| Name | Description | Type | Required | Default |
| :--- | :--- | :--- | :--- | :--- |
| `automerge_test` | Run conftest against the Helm diff output to evaluate if the PR is safe to automerge | boolean | no | `false` |

## Example Output

When changes are detected, a comment is posted to the PR:

> ## Helm chart diff
>
> Rendered manifests compared by resource identity (`apiVersion`, `kind`, `metadata.name`), so resources are matched even when templates move to new paths.
>
> ### `lando/k8s/lando` / `values-dev`
>
> **142 changes across 44 resources** (36 → 41 documents)
>
> **Updated (25)**
> - `apps/v1/Deployment/lando-web`
> - `v1/ServiceAccount/lando`
> - ...
>
> **Added (12)**
> - `v1/Service/lando-web-headless`
> - ...
>
> <details><summary>Show full diff for <code>lando/k8s/lando</code> / <code>values-dev</code></summary>
>
> ```diff
> @@ metadata.annotations.argocd.argoproj.io/sync-wave @@
> # v1/ServiceAccount/lando
> ! ± value change in multiline text (one insert, one deletion)
> - -3
> + -11
> ```
> </details>

Each entry in the full diff is a change block: the path that changed, the resource it belongs to (`# apiVersion/Kind/name`), a description of the change, and the values.

When a config renders identically on both sides it is called out as unchanged rather than omitted silently, so a migration that is intentionally a no-op is visible as one.

If the comment would exceed GitHub's 65536 character limit it is split across several comments, with the summary kept in the first.

## How the manifests are assembled

`render_charts` renders each chart once per values file, then concatenates that config's output into a single multi-document YAML beside the render directory:

```
shared/<ref>-charts/<chart>/<config>/     # helm template --output-dir tree
shared/<ref>-charts/<chart>/<config>.yaml # the same manifests concatenated
```

Both the diff and the automerge evaluation read the `.yaml` file rather than walking the tree themselves. Building it once means the two cannot disagree about what was compared, and the work is not repeated.

Charts are rendered under their **directory name** as the Helm release name, which is what Argo CD uses by default. This matters because the comparison matches resources by name: a release name that differs between the two sides makes every resource appear deleted and re-added. Charts whose tenant config overrides `release_name` are not handled, and the diff flags the symptom when it sees it (see MZCLD-3823).

Concatenation order is irrelevant, since both consumers match documents on resource identity rather than position.

## Automerge evaluation

With `automerge_test: true`, a separate job compares the same concatenated manifests with [diffnest](https://github.com/sters/diffnest) to produce a JSON-patch representation of the change, then runs [conftest](https://www.conftest.dev/) against [`helm-automerge.rego`](https://github.com/mozilla/helm-charts/blob/main/policy/helm-automerge.rego) and reports the result as a `conftest test` commit status.

diffnest is used here rather than dyff because the policy consumes JSON-patch operations and dyff's CLI has no machine-readable output format. The policy only passes when the sole change is to the `mozcloud_chart_version` label, so anything else fails closed.

## Tool versions

| Tool | Version | Used by |
| :--- | :--- | :--- |
| dyff | `v1.12.0` | `diff_helm_charts` |
| diffnest | `v1.7.0` | `evaulate_helm_chart_automerge` |
| conftest | `v0.68.2` | `evaulate_helm_chart_automerge` |
