# Docker Build and Push Reusable Workflow

This reusable GitHub Actions workflow encapsulates an end-to-end process for building and pushing a
container image for a MozCloud service.

The inputs of this workflow are mostly used to configure [`docker-build`] and [`docker-push`].
See their documentation for more details on some of these inputs.

## Overview

- Runs a `prebuild_script` to prepare the build environment (e.g. generate `version.json`)
- Builds and tags the Docker image using [`docker-build`]
- Provides an optional hook to run a `postbuild_script` after building (e.g. for running tests)
- Pushes the image to GCP Artifact Registry (and optionally GitHub Container Registry) using [`docker-push`]

## Inputs

| Name                                    | Required | Type    | Description                                                                                                     |
| --------------------------------------- | -------- | ------- | --------------------------------------------------------------------------------------------------------------- |
| `image_name`                            | true     | string  | The name of the Docker image to build.                                                                          |
| `gar_name`                              | true     | string  | Name of the GAR repository. (typically `<tenant>-prod`)                                                         |
| `project_id`                            | true     | string  | GCP project ID where the Artifact Registry is located. (typically `moz-fx-<tenant>-prod`)                       |
| `workload_identity_pool_project_number` | false    | string  | GCP workload identity pool project number. (default: `${{ vars.GCPV2_WORKLOAD_IDENTITY_POOL_PROJECT_NUMBER }}`) |
| `prebuild_script`                       | false    | string  | Shell script (either inline or path to script) to run before building the image. (default: _see below)_         |
| `postbuild_script`                      | false    | string  | Shell script (either inline or path to script) to run after building the image.                                 |
| `image_build_context`                   | false    | string  | Build context path. Default value is relative to the repository root. (default: `"./"`)                         |
| `dockerfile_path`                       | false    | string  | Path to Dockerfile. Default value is relative to the repository root. (default: `"./Dockerfile"`)               |
| `image_tag_metadata`                    | false    | string  | Additional metadata for image tagging.                                                                          |
| `should_tag_ghcr`                       | false    | boolean | Whether to also tag and push the image to GHCR. (default: `false`)                                              |
| `should_tag_latest`                     | false    | boolean | Whether to tag the image as `latest`. (default: `false`)                                                        |
| `gar_location`                          | false    | string  | Artifact Registry location. (default: `"us"`)                                                                   |
| `service_account_name`                  | false    | string  | GCP service account for pushing to registry. (default: `"artifact-writer"`)                                     |
| `enable_attestations`                   | false    | boolean | Enable SBOM and provenance attestations for supply chain security. (default: `false`)                           |
| `push`                                  | false    | boolean | Set to false to validate build steps without pushing to repository. (default: `true`)                           |

### `enable_attestations`

When enabled, generates SBOM (Software Bill of Materials) and provenance attestations for improved supply chain security:

- **SBOM**: Enables vulnerability scanning and dependency tracking
- **Provenance**: Proves images were built from trusted sources

Attestations are automatically attached to images and preserved when pushed to GAR/GHCR. When enabled, the workflow configures Docker's containerd image store to support generating attestations while maintaining the build → test → push workflow pattern.

**Note**: Enabling attestations switches Docker to use the containerd image store, which [requires additional disk space](https://docs.docker.com/engine/storage/containerd/#disk-space-usage) compared to the default image store. Consider this when running on disk-constrained runners.

### Default `prebuild_script`

```bash
IMAGE_BUILD_CONTEXT_ABSPATH=$(realpath "$IMAGE_BUILD_CONTEXT")

printf '{"commit":"%s","version":"%s","source":"%s","build":"%s"}\n' \
"$GITHUB_SHA" \
"$GITHUB_REF_NAME" \
"$GITHUB_SERVER_URL/$GITHUB_REPOSITORY" \
"$GITHUB_SERVER_URL/$GITHUB_REPOSITORY/actions/runs/$GITHUB_RUN_ID" > "$IMAGE_BUILD_CONTEXT_ABSPATH/version.json"
```

This script generates a `version.json` file with commit, version, and build metadata.

## Best Practices for Prebuild and Postbuild Scripts

### Prebuild Scripts

Prebuild scripts should prepare the build environment and validate inputs before building the container. Common checks include:

- **Linting**: The `docker-build` action used in this workflow uses docker build checks to lint the Dockerfile before building, but you could run additional linting tools here
- **Build metadata generation**: Create version files with commit SHA, build URL, and timestamps (as shown in the default script)
- **Dependency validation**: Verify that lock files are up-to-date and dependencies are properly pinned

### Postbuild Scripts

Postbuild scripts validate the built container image before pushing to registries. Recommended checks include:

- **Smoke tests**: Execute basic functional tests against the running container to verify core functionality
- **Container verification**: Test health check endpoints (`/__heartbeat__`, `/__lbheartbeat__`, `/__version__`) to ensure the service starts correctly
- **Image size validation**: Verify the image size meets expectations to catch bloat from unnecessary dependencies
- **Configuration validation**: Confirm required environment variables and config files are present

For detailed examples and implementation guidance, see:

- [mozilla/cicd-demos/go-demo](https://github.com/mozilla/cicd-demos/tree/main/go-demo) - Reference implementation with prebuild/postbuild scripts
- [How to: Linting and Testing in Container Builds](https://mozilla-hub.atlassian.net/wiki/spaces/SRE/pages/1920860166/How+to+Linting+and+Testing+in+Container+Builds) - Comprehensive guide to container build validation

## Permissions

- `contents: read` (required)
- `id-token: write` (required: for GCP authentication)
- `packages: write` (optional: for GHCR publishing)

## Usage

### Minimal configuration

```yaml
on:
  push:
    branches:
      - main
    tags:
      - v20[0-9][0-9].[01][0-9].[0-3][0-9] # e.g. v2023.12.04
      - v20[0-9][0-9].[01][0-9].[0-3][0-9]-[0-9] # e.g. v2023.12.04-2

jobs:
  build-and-push:
    secrets: inherit
    permissions:
      contents: read
      id-token: write
    uses: mozilla/deploy-actions/.github/workflows/build-and-push.yml@main
    with:
      image_name: your-service
      gar_name: your-tenant-prod
      project_id: moz-fx-you-tenant-prod
```

### Push to GHCR

```yaml
on:
  push:
    branches:
      - main
    tags:
      - v20[0-9][0-9].[01][0-9].[0-3][0-9] # e.g. v2023.12.04
      - v20[0-9][0-9].[01][0-9].[0-3][0-9]-[0-9] # e.g. v2023.12.04-2

jobs:
  build:
    secrets: inherit
    permissions:
      contents: read
      id-token: write
      packages: write
    uses: mozilla/deploy-actions/.github/workflows/build-and-push.yml@main
    with:
      image_name: your-service
      gar_name: your-tenant-prod
      project_id: moz-fx-you-tenant-prod
      should_tag_ghcr: true
```

**Note**: GHCR images are private by default. Depending on which GitHub organization your
repository belongs to, you may need to [file a bug][ghcr-visibility-bug] to allow making
published GHCR images publicly accessible (see [example][ghcr-visibility-example]).

## Troubleshooting

Is your build failing? The "[How to: Publish Container Images to GAR][how-to]"
wiki page may include some answers, specifically around permissions issues.

[`docker-build`]: ../../../docker-build/README.md
[`docker-push`]: ../../../docker-push/README.md
[how-to]: https://mozilla-hub.atlassian.net/wiki/spaces/SRE/pages/997163545/How+to+Publish+Container+Images+to+GAR
[ghcr-visibility-bug]: https://bugzilla.mozilla.org/enter_bug.cgi?product=mozilla.org&component=Github%3A+Administration
[ghcr-visibility-example]: https://bugzilla.mozilla.org/show_bug.cgi?id=2009036
