# CODEOWNERS Linting Reusable Workflow

Validates CODEOWNERS files using the `lint-codeowners` tool found in [mozilla-it/sre-citools](https://github.com/mozilla-it/sre-citools).

## Inputs

| Name           | Required | Type   | Default | Description                                                              |
| -------------- | -------- | ------ | ------- | ------------------------------------------------------------------------ |
| `exclude_dirs` | false    | string | `""`    | Space-separated list of directories to exclude from linting (e.g. `"projects misc modules"`) |

## Usage

```yaml
name: Lint CODEOWNERS

on:
  pull_request:
    paths:
      - 'CODEOWNERS'

jobs:
  lint-codeowners:
    uses: mozilla/deploy-actions/.github/workflows/codeowners.yml@main
    with:
      exclude_dirs: "projects misc modules"  # optional: exclude specific directories
```
