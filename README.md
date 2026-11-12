# maira-code-workflows

Reusable GitHub Actions workflow for automated pull request review with the
Maira Code model (`g-maira-code`).

The reviewer is language- and framework-agnostic. It infers the stack from the
repository, reports only defects introduced by the change, and can run the
project's existing linters, type checkers, and focused tests to validate a
hypothesis. Repositories can append their own review guidance without forking
the workflow.

## What it does

1. **Finding pass** — reads the diff and the surrounding code, and reports at
   most 10 defects that have a concrete failure path.
2. **Verification pass** — adversarially tries to refute each candidate; only
   findings that survive are published.
3. **Publishing** — posts inline comments on lines the diff added, and a
   summary comment for everything else, including severity counts, cost, model,
   and effort.
4. **Cleanup** — resolves this workflow's review threads from the previous run.

## Quick start

Add a workflow to the calling repository:

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
    uses: gigalogy/maira-code-workflows/.github/workflows/pr-review-agent.yml@<full-sha-or-release-tag>
    permissions:
      contents: write        # the resolve job needs this; the caller is the ceiling
      issues: write
      pull-requests: write
    with:
      effort: high
      max_budget_usd: "10"
    secrets:
      MAIRA_CODE_TOKEN: ${{ secrets.MAIRA_CODE_TOKEN }}
```

Pin `uses:` to a full commit SHA or an immutable release tag, never a branch.
The caller chooses the branch filters and event types; the workflow itself only
runs for `pull_request` events.

## Requirements

The calling repository (or its organization) provides two repository variables
and one secret:

| Name | Kind | Example | Purpose |
|---|---|---|---|
| `MAIRA_CODE_BASE_URL` | variable | `https://maira.example.com` | Gateway API root. Do not append `/v1`; the CLI appends `/v1/messages`. |
| `MAIRA_CODE_MODEL` | variable | `g-maira-code` | Model exposed by the gateway. |
| `MAIRA_CODE_TOKEN` | secret | — | Maira Hub API key, sent as the `X-Api-Key` header. |
| `REVIEW_SHOW_COST` | variable | `false` | Set to `false` to hide the cost line in the summary. Shown by default. |

Organization-level variables are inherited by repositories, so they can be set
once for every client repository. Missing values fail the job early with a
message naming the missing item.

## Supported languages and frameworks

The review brief is generic: it reads the manifests and the surrounding code,
follows the repository's conventions, and judges a pattern only by whether it
causes a concrete defect in that stack. No language-specific prompt is needed.

Dependency installation before the review is best effort and driven by the
manifests and lockfiles in `project_directory` (repository root by default):

| Stack | Detected by | Install |
|---|---|---|
| Python | `uv.lock` | `uv sync --frozen` |
| Python | `poetry.lock` | `poetry install` |
| Python | `requirements.txt` | `pip install -r` |
| Python | `pyproject.toml` / `setup.py` | `pip install -e` |
| Node.js / TypeScript | `package-lock.json` | `npm ci` |
| Node.js / TypeScript | `yarn.lock` | `yarn install --frozen-lockfile` |
| Node.js / TypeScript | `pnpm-lock.yaml` | `pnpm install --frozen-lockfile` |
| Node.js / TypeScript | `bun.lockb`, `bun.lock` | `bun install --frozen-lockfile` |
| PHP | `composer.json` | `composer install` |
| Java / Kotlin | `pom.xml` | `mvn dependency:go-offline` |
| Java / Kotlin | `gradlew`, `build.gradle(.kts)` | `gradle dependencies` |
| Go | `go.mod` | `go mod download` |
| Rust | `Cargo.toml` | `cargo fetch` |
| Ruby | `Gemfile` | `bundle install` |
| .NET | `*.csproj`, `*.sln` | `dotnet restore` |

Notes:

- A failed install is a warning, not a failure; the review still runs from
  source. Set `install_dependencies: false` for stacks not listed here.
- In a monorepo, set `project_directory` to the directory holding the app's
  manifest. The diff is always taken across the whole repository.
- `python_version` is only used when a Python project is detected.
- The agent may run repository-defined lint, type-check, and test commands for
  the stacks above. The command allowlist is deliberately narrow and blocks
  network clients, package installs, and file edits. When a check needs a
  command that is not allowed, the agent is instructed to say so rather than
  work around it.

## Inputs

| Input | Default | Description |
|---|---|---|
| `effort` | `high` | Claude effort level for both passes. |
| `max_budget_usd` | `"10"` | Spending ceiling for one pass, shared across retry attempts. |
| `python_version` | `"3.11"` | Python to install when a Python project is detected. Ignored otherwise. |
| `project_directory` | `"."` | Directory containing the app and its manifest(s), relative to the repository root. |
| `install_dependencies` | `true` | Install the project environment before reviewing. Set `false` for unsupported stacks. |
| `prompt_repository` | `""` | Repository containing an additional, repository-specific review brief. |
| `prompt_ref` | `""` | Full 40-character commit SHA of `prompt_repository`. Required when `prompt_repository` is set. Use the PR's base SHA. |
| `prompt_path` | `.github/maira/pr-review.md` | Repository-specific brief, relative to `prompt_repository`. Appended to the default brief. |
| `verify_prompt_path` | `prompts/pr-review-verify.md` | Verification brief, relative to this repository. Not overridable. |
| `schema_path` | `prompts/pr-review.schema.json` | Output JSON schema, relative to this repository. Not overridable. |

## Secrets

| Secret | Required | Description |
|---|---|---|
| `MAIRA_CODE_TOKEN` | yes | Maira Hub API key, sent as the `X-Api-Key` header. |
| `WORKFLOWS_TOKEN` | no | Read access to this repository when it is private, or to a separate private `prompt_repository`. Not needed when every source repository is public, or when the brief comes from the caller's own repository. Without it the job uses `github.token`, which cannot read other private repositories. |

## Repository-specific review guidance

The default brief is intentionally generic. A repository can append its own
guidance — stack, layout, domain invariants, what is critical — without
maintaining a fork of the workflow.

Create a Markdown file in the calling repository, for example
`.github/maira/pr-review.md`:

```markdown
### Stack

Laravel 11 API with MySQL, Redis queues, and a Vue 3 admin panel.

### Layout

- `app/` domain code; `app/Http/Controllers` stays thin.
- `database/migrations` is append-only; never edit a released migration.

### Invariants that matter

- Every query on a tenant-owned table must be scoped through
  `TenantScope`; a missing scope is a critical finding.
- Jobs must be idempotent: `ShouldBeUnique` or an explicit dedupe key.
- Money is stored in minor units as integers; never introduce floats.
- Admin mutations must write an audit record.

### Out of scope

- `resources/js/components/ui/` is vendored from the design system.
- Generated files under `bootstrap/cache/`.
```

Then call the workflow with the brief pinned to the PR's **base** SHA:

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

Rules and caveats:

- `prompt_ref` must be a full 40-character commit SHA; branches and tags are
  rejected. Use the base SHA, never the head or a branch: the base commit
  predates the pull request, so a pull request cannot rewrite the guidance that
  reviews it.
- The brief is **appended** to the default brief. The default brief keeps the
  severity scale, untrusted-input rules, and output contract, so repository
  guidance can add context but cannot weaken them.
- The brief must already exist in the base commit. Land it in a separate merged
  pull request before enabling the reviewer, otherwise the first run fails with
  `missing from <repo>@<sha>`.
- When the brief comes from the caller's own repository, `github.token` can read
  it even when private, so `WORKFLOWS_TOKEN` is not required.
- `prompt_repository` and `prompt_ref` apply only to the repository brief. The
  verification prompt and output schema always come from this repository and
  stay in lockstep with the extraction and publishing code.

## Security model

- The job only runs for same-repository, non-draft, human-authored pull
  requests. Fork pull requests never receive the token.
- The review job uses `contents: read`, and the agent's command allowlist blocks
  file edits, commits, pushes, network clients (`curl`, `wget`), package
  installs, and `gh`. Build tools it is allowed to run can still reach their
  package registries.
- Pull request titles, descriptions, diffs, comments, and file contents are
  treated as untrusted data, and the agent is told to report prompt injection
  instead of following it.
- Resolving review threads requires `contents: write`, so it happens in a
  separate job with no checkout and no untrusted-content processing.
- The gateway key is scrubbed from everything published, and GitHub masks the
  secret in logs.
- Repository build and test scripts still execute with the token present in the
  job environment. Only collaborators who can push branches can trigger those
  paths; use protected branches so untrusted contributors cannot create
  in-repository branches.

## Cost and limits

- `max_budget_usd` caps one pass and is shared across retry attempts.
- Transient gateway failures (`429`, `5xx`, Cloudflare `524`) are retried once
  after 120 seconds.
- Each job has a 40-minute timeout and publishes at most 10 findings.
- Diffs larger than 400 KB are truncated in the prompt; the agent is told to
  read the remaining files directly.

## Review behaviour

- Each run posts a new review and a summary comment (updated in place on later
  runs), then resolves this workflow's inline threads from the previous run.
- Inline comments are only placed on lines the diff added; everything else goes
  into the summary.
- Check names are prefixed with the caller job name, for example
  `review / review-agent`.
- The caller's base commit must be available in the checkout; the workflow
  fetches it explicitly.

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `Repository variable MAIRA_CODE_BASE_URL is not set` | Set the variable on the calling repository or its organization. |
| `unrecognized_model` | `MAIRA_CODE_MODEL` is missing or wrong; it must match the model exposed by the gateway. |
| `prompt_ref must be a full 40-character commit SHA` | Use `${{ github.event.pull_request.base.sha }}`, not a branch or tag. |
| `... is missing from <repo>@<sha>` | The brief or prompt file does not exist at that commit. Land it first. |
| Dependency install warning | The stack is unsupported or the install failed; the review continues. Set `install_dependencies: false` to silence it. |
| Inline review failed, findings folded into the summary | GitHub rejected a comment position; the finding is still published in the summary. |
| No review posted | The PR is a draft, authored by a bot, or from a fork; the gate skipped it. |

## Versioning and releases

- Consumers should pin `uses:` to a full commit SHA from a release page.
- Tags in the workflows repository are the upgrade boundary. Protect release
  tags and `main` so a tag cannot be moved to unreviewed code: every caller
  passes its gateway key into this workflow.
- To upgrade, update the pinned SHA in the caller workflow; no other change is
  required.

## Layout

```
.github/workflows/pr-review-agent.yml
prompts/pr-review.md
prompts/pr-review-verify.md
prompts/pr-review.schema.json
```

## License

[MIT](LICENSE)
