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
| `prompt_repository` | `""` | Repository holding the prompt files. Defaults to this repository. |
| `prompt_ref` | `""` | Ref of `prompt_repository`. Defaults to this workflow's commit; required when `prompt_repository` is set. |
| `prompt_path` | `prompts/pr-review.md` | Review brief, relative to `prompt_repository`. |
| `verify_prompt_path` | `prompts/pr-review-verify.md` | Verification brief, relative to `prompt_repository`. |
| `schema_path` | `prompts/pr-review.schema.json` | Output JSON schema, relative to `prompt_repository`. |

## Secrets

| Secret | Required | Description |
|---|---|---|
| `MAIRA_CODE_TOKEN` | yes | Maira Hub API key, sent as the `X-Api-Key` header. |
| `WORKFLOWS_TOKEN` | no | Token with read access to `prompt_repository`. Only needed when this repository is private. Without it the job uses `github.token`, which cannot read other private repositories. |

## Repository variables

The caller's variables configure the gateway:

| Variable | Example | Description |
|---|---|---|
| `MAIRA_CODE_BASE_URL` | `https://maira.example.com` | Gateway API root. Do not append `/v1`; the CLI appends `/v1/messages`. |
| `MAIRA_CODE_MODEL` | `g-maira-code` | Model exposed by the gateway. |
| `REVIEW_SHOW_COST` | `false` | Set to `false` to hide the cost line in the review summary. Shown by default. |

Organization-level variables are inherited by repositories, so these can be set once at the organization level.

## Prompts

The reviewer loads its brief, verification brief, and output schema from this repository at the called workflow's commit. The prompt files are therefore trusted by construction and identical across callers.

To use a repository-specific brief, set `prompt_repository` and `prompt_ref` to a repository and immutable ref that contain the three files. If you point them at the caller itself, use the base commit SHA so a pull request cannot rewrite the reviewer's instructions.

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
