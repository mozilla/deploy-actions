# GitHub Deployment/Release Script

This action is used to set the deployment status for GitHub deployments on a different repository.

ex. https://github.com/mozilla/fxa/deployments & https://github.com/mozilla/fxa/releases

## Inputs

### `github_org`

The GitHub organization hosting the target repository. Default `"mozilla"`

### `repository`

**Required** The GitHub repository to target.

### `environment_url`

The environment URL to set in the deployment request.

### `state`

The deployment state. Supported values in_progress, success, failure. Default `"in_progress"`

### `deployment_id`

The id of the pre-existing GitHub deployment to set the status for.

### `environment`, `ref`, `sha`

Used to uniquely identify the deployment to update, as an alternative to `deployment_id`. Will be ignored if `deployment_id` is set. If no deployment with the given settings exists, a new deployment will be created using this information.

### `app_id`

The id of the GitHub app with the permission to update deployment statuses on the target repo.

### `private_key`

The private key of the GitHub app.

### Example usage
```
name: github-deployment-update

on:
  workflow_dispatch:
    inputs:
      state:
        type: string
        default: "in_progress"
      environment:
        description: "Target deployment environment"
        type: string
        default: "staging"
      ref:
        description: "Deployed git ref"
        required: true
        default: "v1.269.2"

jobs:
  update_deployment_job:
    runs-on: ubuntu-latest
    steps:
      - name: Set repository deployment status
        uses: mozilla-it/deploy-actions/deployment-status@v4.0.0
        with:
          github_org: mozilla
          repository: fxa
          environment_url: "https://accounts.stage.mozaws.net/__version__"
          state: ${{ inputs.state }}
          environment: ${{ inputs.environment }}
          ref: ${{ inputs.ref }}
          app_id: ${{ secrets.APPLICATION_ID }}
          private_key: ${{ secrets.PRIVATE_KEY }}
```


### Required Secrets

A GitHub app with the permissions to create/modify deployments and/or releases on the target repositories must be created to use this function. The following secrets are also required to be passed in as environment variables:

- `APPLICATION_ID` - The application ID of the required GitHub app

- `PRIVATE_KEY` - The private key of the required GitHub app
