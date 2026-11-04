# Claude Review Reusable Workflow

Reviews a pull request with Claude and posts one comment. It reads the PR diff plus whatever
change previews already exist on the PR:

- **Atlantis** plan comments (`Ran Plan for ...`, `Plan: X to add, Y to change, Z to destroy`,
  lock and unlock notices)
- **Spacelift** run comments posted by the notification policy (change summary per stack, and
  the planning log when the policy includes it)
- The **Helm diff** from [`diff-rendered-charts`](./diff-rendered-charts.md), read from the
  artifact it uploads or from its PR comment
- kubeconform validation failures and commit statuses / check runs

Claude compares what the plan and diff *do* against what the PR *says* and reports unexpected
destroys or replacements, out-of-scope changes, risky Helm changes (probes, resources, env vars,
selectors, missing workloads), and whether the change is ready to apply.

This is the Cloud Engineering review for infrastructure repositories. For a generic code
review of application repositories see
[`mozilla/actions/claude-review`](https://github.com/mozilla/actions#claude-review--claude-code-review).

## How it runs

Four jobs:

| Job | Token | What it does |
|---|---|---|
| `gate` | read + `pull-requests: write` | Decides whether the event is a review trigger, resolves the PR, refuses forks and closed PRs. Runs in seconds; non-trigger comments never start a runner. When a human asks on a closed PR it replies saying so, instead of silently skipping. |
| `review` | read-only (+ `checks`, `statuses`) | Collects plan comments, Spacelift comments, Helm diff, checks and discussion into files, then runs `anthropics/claude-code-action` with `Read`, `Grep`, `Glob`, `gh pr diff` and `gh pr view` only. Claude cannot write to GitHub or the workspace. Returns schema-validated structured output. |
| `post` | write | Minimizes earlier review comments, posts exactly one new one, optionally sets a commit status. A failed review still produces a comment so a silent run never reads as a clean one. |
| `health` | caller's | Schedule only. Audits the `post` job's conclusions over the last 7 days (it fails whenever no review was produced) and fails (optionally posting to Slack) when reviews are failing or the workflow is not starting. |

The `review` job has a per-PR concurrency group with `cancel-in-progress`, so the newest plan
comment or request supersedes a running review. Only review triggers reach that job, so an
unrelated comment (including your own `atlantis plan`) cannot cancel a review in flight.

Triggers, by caller event:

| Caller event | When the review runs |
|---|---|
| `issue_comment` | A human with write access (owner, member or collaborator) posts a comment containing the trigger phrase (default `@claude`), or (in `auto` mode) Atlantis posts `Ran Plan ...`, or Spacelift posts or edits a run comment. Subscribe to `created` and `edited`; edits by humans are ignored, and so are the `edited` events Atlantis raises when it minimizes its previous plan comments |
| `pull_request` | On the event (use `opened`, `ready_for_review`); diff-only review when no plan exists yet. Bot-authored and draft PRs are skipped here but still get plan-triggered reviews |
| `workflow_dispatch` | Pass `pr_number` |
| `schedule` | No review. Runs the `health` job instead: audits the last 7 days of this workflow's runs in the calling repo and fails when any ended in `startup_failure`, when every review produced nothing, or when failed `post` jobs outnumber successful ones (gate-only runs are not counted). With the optional `SLACK_WEBHOOK_URL` secret it also posts to Slack. The caller must grant `actions: read` |

If the `ANTHROPIC_API_KEY` secret is empty the gate skips with a notice, so a repository can adopt
the caller before the secret exists without PR noise.

## Inputs

| Name | Type | Default | Description |
|---|---|---|---|
| `mode` | string | `auto` | `auto` reviews on plan comments and the trigger phrase; `on-demand` only on the trigger phrase. Neither affects `pull_request` or `workflow_dispatch` callers |
| `trigger_phrase` | string | `@claude` | Comment text from a human that requests a review |
| `atlantis_bot` | string | `atlantis-admin[bot]` | Login of the Atlantis GitHub App in this repo (`vars.ATLANTIS_GITHUB_USER` where defined) |
| `spacelift_bot` | string | `spacelift-io[bot]` | Login of the Spacelift GitHub App |
| `helm_diff_artifact` | string | `""` | `diff_artifact` output of `diff-rendered-charts` in the same run |
| `extra_prompt` | string | `""` | Repository-specific review instructions |
| `model` / `fallback_model` | string | `""` | Claude models; empty uses the action defaults |
| `budget_usd` | string | `6.00` | Spend cap per review |
| `debounce_seconds` | string | `45` | Wait after a plan comment so continuation comments and sibling stacks land first |
| `post_status` | boolean | `false` | Publish a `claude-review` commit status (caller needs `statuses: write`) |
| `pr_number` | string | `""` | Explicit PR number for `workflow_dispatch` callers |
| `paths` / `paths_ignore` | string | `""` | Newline-separated globs applied to the PR's changed files on every trigger (plan comments carry no paths). `paths` requires a match; `paths_ignore` skips PRs whose files all match. Bash pattern semantics: `*` also matches `/`, so `dir/*.yaml` covers subdirectories |
| `comment_marker` | string | `claude-review` | Identifier for this caller's comments and concurrency group. Give each caller on a repo its own |

Secrets: `ANTHROPIC_API_KEY` (required), `SLACK_WEBHOOK_URL` (optional, health check only).

## Permissions

The reusable workflow downgrades the `review` job to `contents: read`, `pull-requests: read`,
`checks: read`, `statuses: read` (the last two let it read Atlantis plan statuses and CI results);
the `gate` job declares `contents: read`, `pull-requests: write` (for its closed-PR reply); the
`post` and `health` jobs inherit the caller's grant. GitHub validates every called job's
declared permissions against the caller's grant at startup, including jobs that `if:` skips, which
is why only the minimum is declared inside. Callers must grant at least:

```yaml
permissions:
  contents: read
  pull-requests: write
  checks: read
  statuses: read    # statuses: write with post_status: true
  actions: read     # with a schedule trigger (health check)
```

Humans who comment the trigger phrase must be a repository owner, member or collaborator (the
`author_association` GitHub reports). Comments from anyone else are not a trigger: no job runs
and nothing is posted.

## Usage

### Plan-triggered and on-demand reviews (Atlantis / Spacelift repos)

The examples use `@main` for brevity. Real callers should pin a release tag (`vX.Y.Z`); this
workflow is not part of v6.8.0 or earlier.

```yaml
# .github/workflows/claude-review.yaml
name: Claude Review
on:
  issue_comment:
    types: [created, edited]           # edited: Spacelift updates its plan comments in place
  pull_request:
    types: [opened, ready_for_review]   # optional: diff-only review before a plan exists
  schedule:
    - cron: "0 13 * * 1-5"             # weekday health check

# Do not add a PR-scoped concurrency group with cancel-in-progress here: the reusable
# workflow scopes its own group to actual review runs.

jobs:
  claude-review:
    uses: mozilla/deploy-actions/.github/workflows/claude-review.yml@main
    permissions:
      contents: read
      pull-requests: write
      checks: read
      statuses: read
      actions: read
    with:
      atlantis_bot: ${{ vars.ATLANTIS_GITHUB_USER }}
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

### After the Helm diff

Add a job to the workflow that calls `diff-rendered-charts`:

```yaml
jobs:
  diff:
    uses: mozilla/deploy-actions/.github/workflows/diff-rendered-charts.yml@main
    permissions:
      contents: read
      pull-requests: write

  claude-review:
    needs: diff
    if: needs.diff.outputs.changes_found == 'true'
    uses: mozilla/deploy-actions/.github/workflows/claude-review.yml@main
    permissions:
      contents: read
      pull-requests: write
      checks: read
      statuses: read
    with:
      helm_diff_artifact: ${{ needs.diff.outputs.diff_artifact }}
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

### Using the composite actions directly

Supply your own trigger, gate and permissions, keep review and posting in separate jobs, and
check out the PR head before the review step:

```yaml
jobs:
  review:
    runs-on: ubuntu-latest
    permissions: { contents: read, pull-requests: read, checks: read, statuses: read }
    outputs: { review: ${{ steps.review.outputs.review }} }
    steps:
      - uses: actions/checkout@<sha>
        with: { ref: refs/pull/${{ github.event.issue.number }}/head, persist-credentials: false }
      - id: review
        uses: mozilla/deploy-actions/claude-review@<sha>
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          pr_number: ${{ github.event.issue.number }}
  post:
    needs: review
    if: ${{ !cancelled() }}
    runs-on: ubuntu-latest
    permissions: { pull-requests: write }
    steps:
      - uses: mozilla/deploy-actions/claude-review/post@<sha>
        with:
          review: ${{ needs.review.outputs.review }}
          pr_number: ${{ github.event.issue.number }}
```

## Example output

```markdown
### Claude review: Plan destroys 2 resources the diff does not touch.

**Status:** findings  |  **Reviewed:** Atlantis plan: 2 projects, PR diff  |  **Missing:** Helm diff

#### Findings
- **:red_circle: high** · `terraform-plan` · **Forced replacement of the prod Cloud SQL instance**
  The plan replaces `google_sql_database_instance.main` in `dataservices-high/tf/prod` because
  `settings.tier` changed, but the PR only adds a read replica.
  `google_sql_database_instance.main`, `# forces replacement`

#### Apply readiness
- Plans present for both touched projects; no lock contention.
- `dataservices-high/tf/nonprod` will not autodeploy on merge.

<sub>trigger: atlantis-plan · model: default · $1.84 · 23 turns · 96s · run</sub>
```

## Commit status semantics

With `post_status: true` the `claude-review` status is `success` when a review was produced
(findings or not) and `failure` when none could be produced. Making it a required check therefore
means "Claude reviewed this", never "Claude approved this". Humans decide what blocks.

## Pinning and the org allowlist

Only `anthropics/claude-code-action@v1` is on the Mozilla organization's Actions allowlist, so the
pin lives in this repository alone, with a Dependabot ignore rule. Callers reference
`mozilla/deploy-actions/...@vN`, which is always allowed within the organization.

## Troubleshooting

- **Nothing happened after `@claude`.** Check the caller run: the `gate` job prints why it skipped
  (closed PR, fork, not a trigger). Closed PRs get a reply; forks do not.
- **Review cancelled.** A newer plan comment or request on the same PR superseded it. The newer run
  posts the review.
- **"No review produced".** Usually the budget cap. Raise `budget_usd` or narrow `extra_prompt`.
- **Every review fails with "no review produced".** Most likely the `anthropics/claude-code-action`
  ref is no longer `@v1` or the org allowlist changed; because the action is resolved inside the
  composite, this shows as a failed `post` job rather than `startup_failure`. Fix it here once;
  every caller recovers. The scheduled health check reports it as reviews that "produced nothing
  and none succeeded".
- **Spacelift plans not reviewed.** The Spacelift notification policy must post run comments on
  the PR (see `spacelift/tf/root/policies/notification/` in global-platform-admin); the review
  reads comments, it does not call the Spacelift API.
