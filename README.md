# Mozilla Deployment GitHub Actions

This repository contains GitHub Actions Composite Actions used for Deployment Automation.

### Versioning and Releases

Versioning is automated based on [Semantic Versioning](https://semver.org/) using [`semantic-release`](https://github.com/semantic-release/semantic-release).
Release changelogs are automated by enforcing [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/)
as a PR check using [`semantic-pull-request`](https://github.com/marketplace/actions/semantic-pull-request).

Conventional commit convention will be checked on:
* commit message for **PRs with a single commit**
* PR title for **PRs with multiple commits**

> #### 💡 Tip
>
> Push an empty commit to force `Semantic PR` check on the PR title instead of the commit message if `Semantic PR`
> GitHub Action prevents merging because a commit message does not respect the Conventional Commits specification.
> ```shell
> git commit --allow-empty -m "Semantic PR check"
> ```


Additionally, commit squashing is required before merging for PRs with multiple commits.

#### Release rules matching
From [`semantic-release/commit-analyzer`](https://github.com/semantic-release/commit-analyzer):

- Commits with a breaking change will be associated with a `major` release.
- Commits with `type` 'feat' will be associated with a `minor` release.
- Commits with `type` 'fix' will be associated with a `patch` release.
- Commits with `type` 'perf' will be associated with a `patch` release.
- Commits with scope `no-release` will not be associated with a release type even if they have a breaking change or the `type` 'feat', 'fix' or 'perf'.
- Commits with `type` 'style' will not be associated with a release type.
- Commits with `type` 'test' will not be associated with a release type.
- Commits with `type` 'chore' will not be associated with a release type.


#### Valid commit messages and PR titles :
The tables below shows which commit message or PR title gets you which release type when `semantic-release` runs (using the default configuration):

| PR title / commit message                                                                                                                                                                        | Release type                                                                                                                                |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| `fix: GKE bastion host default notes.`                                                                                                                                                           | ~~Patch~~ Fix Release                                                                                                                       |
| `feat: Copy google-cdn-external from cloudops-infra.`                                                                                                                                            | ~~Minor~~ Feature Release                                                                                                                   |
| `feat(google_cloudsql_mysql): Add query insights settings.`                                                                                                                                      | ~~Minor~~ Feature Release                                                                                                                   |
| `refactor!: Drop support for Terraform 0.12.`                                                                                                                                                    | ~~Major~~ Breaking Release <br /> (Note that since PR titles only have a single line, you have to use the `!` syntax for breaking changes.) |
| `perf(pencil): remove graphiteWidth option`<br><br>`BREAKING CHANGE: The graphiteWidth option has been removed.`<br>`The default graphite width of 10mm is always used for performance reasons.` | ~~Major~~ Breaking Release <br /> (Note that the `BREAKING CHANGE: ` token must be in the footer of the commit message)                     |

### Release Tags

Tags additionally provide "generic" versions (e.g. `v2`, `v3`) to allow specifying "moving" pointers in github actions (e.g. `mozilla/deploy-actions/slack@v3`. This is currently handled manually.
