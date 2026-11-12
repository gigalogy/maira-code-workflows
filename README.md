# maira-code-workflows

Reusable GitHub Actions workflows for Maira Code automated PR reviews.

## Workflows

| Workflow | Purpose |
|---|---|
| `.github/workflows/pr-review-agent.yml` | Claude-based PR review: finding pass, verification pass, inline comments, cost reporting, and stale-thread resolution. |

## Usage

Call the reviewer from a repository workflow:

```yaml
name: PR Review
on:
  pull_request:
    branches: [develop]
    types: [opened, reopened, synchronize, ready_for_review, review_requested]

concurrency:
  group: pr-review-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true

jobs:
  review:
    uses: gigalogy/maira-code-workflows/.github/workflows/pr-review-agent.yml@<full-sha>
    permissions:
      contents: write        # the resolve job needs it; the caller is the ceiling
      issues: write
      pull-requests: write
    with:
      effort: high
      max_budget_usd: "10"
    secrets:
      MAIRA_CODE_TOKEN: ${{ secrets.MAIRA_CODE_TOKEN }}
```

Pin `uses:` to a full commit SHA (recommended) or a release tag, never a branch.

## Inputs

| Input | Default | Description |
|---|---|---|
| `effort` | `high` | Claude effort level for both passes. |
| `max_budget_usd` | `"10"` | Spending ceiling for one pass, shared across retry attempts. |
| `python_version` | `"3.11"` | Python used to set up the project environment. |
| `project_directory` | `code` | Directory containing the application and its lockfile. |
| `install_dependencies` | `true` | Install the project environment before reviewing. Set `false` for non-Python repos. |
| `prompt_repository` | `""` | Repository holding the review brief. Defaults to this repository. |
| `prompt_ref` | `""` | Full 40-character commit SHA of `prompt_repository`. Required when `prompt_repository` is set. Pin the pull request's base SHA. |
| `prompt_path` | `prompts/pr-review.md` | Review brief, relative to the brief source. |
| `verify_prompt_path` | `prompts/pr-review-verify.md` | Verification brief, relative to this repository. Not overridable. |
| `schema_path` | `prompts/pr-review.schema.json` | Output JSON schema, relative to this repository. Not overridable. |

## Secrets

| Secret | Required | Description |
|---|---|---|
| `MAIRA_CODE_TOKEN` | yes | Maira Hub API key, sent as the `X-Api-Key` header. |
| `WORKFLOWS_TOKEN` | no | Token with read access to this repository when it is private, or to a separate private `prompt_repository`. Not needed when the brief comes from the caller's own repository, or when every source repository is public. Without it the job uses `github.token`, which cannot read other private repositories. |

## Repository variables

The caller's variables configure the gateway:

| Variable | Example | Description |
|---|---|---|
| `MAIRA_CODE_BASE_URL` | `https://maira.example.com` | Gateway API root. Do not append `/v1`; the CLI appends `/v1/messages`. |
| `MAIRA_CODE_MODEL` | `g-maira-code` | Model exposed by the gateway. |
| `REVIEW_SHOW_COST` | `false` | Set to `false` to hide the cost line in the review summary. Shown by default. |

Organization-level variables are inherited by repositories, so these can be set once at the organization level.

## Prompts

By default the reviewer loads all three files from this repository at the called workflow's commit. They are trusted by construction and identical across callers.

### Repository-specific review brief

Only the review brief is overridable. Set `prompt_repository` to the caller and `prompt_ref` to the pull request's **base** SHA:

```yaml
jobs:
  review:
    uses: gigalogy/maira-code-workflows/.github/workflows/pr-review-agent.yml@<full-sha>
    permissions:
      contents: write
      issues: write
      pull-requests: write
    with:
      prompt_repository: ${{ github.repository }}
      prompt_ref: ${{ github.event.pull_request.base.sha }}
      prompt_path: .github/maira/pr-review.md
    secrets:
      MAIRA_CODE_TOKEN: ${{ secrets.MAIRA_CODE_TOKEN }}
```

`prompt_ref` must be a full 40-character commit SHA; branches and tags are rejected. Use the base SHA, never the head or a branch: the base commit predates the pull request, so a pull request cannot rewrite the instructions that review it. A branch or tag could also move between the calling workflow and the prompt checkout.

Two other consequences:

- The verification prompt and output schema always come from this repository. They are coupled to the workflow's extraction and publishing code, so a repository-local copy could silently drop findings.
- The brief must already exist in the base commit. Land it in a separate merged pull request before enabling the reviewer, otherwise the first run fails with "missing from `<repo>@<sha>`".

When the brief comes from the caller's own repository, `github.token` has read access even for a private repository, so `WORKFLOWS_TOKEN` is not required.

## Behaviour notes

- The job only runs for same-repository, non-draft, human-authored pull requests; fork PRs do not receive secrets.
- The caller must grant `contents: write`: `resolveReviewThread` is gated on it. The agent job itself still runs with `contents: read`; resolution happens in a separate job with no checkout.
- Each run posts a new review and then resolves the threads from the previous run.
- Job check names are prefixed with the caller job name, e.g. `review / review-agent`.
- The caller's base commit must be available in the checkout; the workflow fetches it.

## Layout

```
.github/workflows/pr-review-agent.yml
prompts/pr-review.md
prompts/pr-review-verify.md
prompts/pr-review.schema.json
```

## License

[MIT](LICENSE)
