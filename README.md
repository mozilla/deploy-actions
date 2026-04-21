# Mozilla Deployment GitHub Actions

This repository contains GitHub Actions Composite Actions used for Deployment Automation.

## Actions
* [`docker-build`](./docker-build/README.md)
* [`docker-push`](./docker-push//README.md)

## Workflows
* [build-and-push](./.github/workflows/docs/build-and-push.md)
* [codeowners](./.github/workflows/docs/codeowners.md)
* [diff-rendered-charts](./.github/workflows/docs/diff-rendered-charts.md)
* [psa-checker](./.github/workflows/docs/psa-checker.md)
* [validate-k8s-manifests](./.github/workflows/docs/validate-k8s-manifests.md)


## Releases & Tags

Releases follow semantic versioning. Tags additionally provide "generic" versions (e.g. `v2`, `v3`) to allow specifying "moving" pointers in github actions (e.g. `mozilla/deploy-actions/slack@v3`.
