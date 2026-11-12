# maira-code-workflows

Reusable GitHub Actions workflow for automated pull request review with the
Maira Lens reviewer (model `g-maira-code`).

The reviewer is language- and framework-agnostic. It infers the stack from the
repository, reports only defects introduced by the change, and can run the
project's existing linters, type checkers, and focused tests to validate a
hypothesis. Repositories can append their own review guidance without forking
the workflow.

## Setup

For a step-by-step organization setup guide you can share with clients, see
[docs/github-organization-setup.md](docs/github-organization-setup.md). It
covers Actions policy, organization variables and secrets, the caller
workflow, optional GitHub App branding, repository-specific briefs, and
troubleshooting.

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
      MAIRA_LENS_TOKEN: ${{ secrets.MAIRA_LENS_TOKEN }}
```

Pin `uses:` to a full commit SHA or an immutable release tag, never a branch.
The caller chooses the branch filters and event types; the workflow itself only
runs for `pull_request` events.

The `MAIRA_LENS_*` names and the `app_id` input require a release after
`v1.0.0`; the `v1.0.0` tag uses the older `MAIRA_CODE_*` names.

## Requirements

The calling repository (or its organization) provides the following repository
variables and secret:

| Name                   | Kind     | Example                     | Purpose                                                                |
|------------------------|----------|-----------------------------|------------------------------------------------------------------------|
| `MAIRA_LENS_BASE_URL`  | variable | `https://maira.example.com` | Gateway API root. Do not append `/v1`; the CLI appends `/v1/messages`. |
| `MAIRA_LENS_MODEL`     | variable | `g-maira-code`              | Model exposed by the gateway.                                          |
| `MAIRA_LENS_TOKEN`     | secret   | —                           | Maira Hub API key, sent as the `X-Api-Key` header.                     |
| `MAIRA_LENS_SHOW_COST` | variable | `false`                     | Set to `false` to hide the cost line in the summary. Shown by default. |

Repositories inherit organization-level variables, so they can be set
once for every client repository. Missing values fail the job early with a
message naming the missing item.

## Supported languages and frameworks

The review brief is generic: it reads the manifests and the surrounding code,
follows the repository's conventions, and judges a pattern only by whether it
causes a concrete defect in that stack. No language-specific prompt is needed.

Dependency installation before the review is the best effort and driven by the
manifests and lockfiles in `project_directory` (repository root by default):

| Stack                | Detected by                     | Install                          |
|----------------------|---------------------------------|----------------------------------|
| Python               | `uv.lock`                       | `uv sync --frozen`               |
| Python               | `poetry.lock`                   | `poetry install`                 |
| Python               | `requirements.txt`              | `pip install -r`                 |
| Python               | `pyproject.toml` / `setup.py`   | `pip install -e`                 |
| Node.js / TypeScript | `package-lock.json`             | `npm ci`                         |
| Node.js / TypeScript | `yarn.lock`                     | `yarn install --frozen-lockfile` |
| Node.js / TypeScript | `pnpm-lock.yaml`                | `pnpm install --frozen-lockfile` |
| Node.js / TypeScript | `bun.lockb`, `bun.lock`         | `bun install --frozen-lockfile`  |
| PHP                  | `composer.json`                 | `composer install`               |
| Java / Kotlin        | `pom.xml`                       | `mvn dependency:go-offline`      |
| Java / Kotlin        | `gradlew`, `build.gradle(.kts)` | `gradle dependencies`            |
| Go                   | `go.mod`                        | `go mod download`                |
| Rust                 | `Cargo.toml`                    | `cargo fetch`                    |
| Ruby                 | `Gemfile`                       | `bundle install`                 |
| .NET                 | `*.csproj`, `*.sln`             | `dotnet restore`                 |

Notes:

- A failed installation is a warning, not a failure; the review still runs from
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

| Input                  | Default                         | Description                                                                                                            |
|------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------------|
| `effort`               | `high`                          | Claude effort level for both passes.                                                                                   |
| `max_budget_usd`       | `"10"`                          | Spending ceiling for one pass, shared across retry attempts.                                                           |
| `python_version`       | `"3.11"`                        | Python to install when a Python project is detected. Ignored otherwise.                                                |
| `project_directory`    | `"."`                           | Directory containing the app and its manifest(s), relative to the repository root.                                     |
| `install_dependencies` | `true`                          | Install the project environment before reviewing. Set `false` for unsupported stacks.                                  |
| `app_id`               | `""`                            | Optional GitHub App ID used to author review feedback. Leave empty to post as `github-actions[bot]`.                   |
| `prompt_repository`    | `""`                            | Repository containing an additional, repository-specific review brief.                                                 |
| `prompt_ref`           | `""`                            | Full 40-character commit SHA of `prompt_repository`. Required when `prompt_repository` is set. Use the PR's base SHA.  |
| `prompt_path`          | `.github/maira/pr-review.md`    | Repository-specific brief, relative to `prompt_repository`. Combined with the default brief according to `brief_mode`. |
| `brief_mode`           | `append`                        | How the repository-specific brief combines with the default brief: `append` or `replace`.                              |
| `verify_prompt_path`   | `prompts/pr-review-verify.md`   | Verification brief, relative to this repository. Not overridable.                                                      |
| `schema_path`          | `prompts/pr-review.schema.json` | Output JSON schema, relative to this repository. Not overridable.                                                      |

## Secrets

| Secret                       | Required | Description                                                                                                                                                                                                                                                                                         |
|------------------------------|----------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `MAIRA_LENS_TOKEN`           | yes      | Maira Hub API key, sent as the `X-Api-Key` header.                                                                                                                                                                                                                                                  |
| `MAIRA_LENS_APP_PRIVATE_KEY` | no       | Private key for `app_id`. Required when `app_id` is set.                                                                                                                                                                                                                                            |
| `MAIRA_LENS_WORKFLOWS_TOKEN` | no       | Read access to this repository when it is private, or to a separate private `prompt_repository`. Not needed when every source repository is public, or when the brief comes from the caller's own repository. Without it the job uses `github.token`, which cannot read other private repositories. |

## Branding

The review content is branded by default: the summary is titled
`Maira Lens review`, inline comments carry a `Maira Lens · confidence` footer,
and the workflow appears as `Maira Lens PR Review` with a `Maira Lens review`
job.

The **author identity** on comments is controlled by the token that posts them:

- With no extra configuration the workflow uses `GITHUB_TOKEN`, which GitHub
  always displays as `github-actions[bot]`. Its name and avatar cannot be
  changed.
- To post as Maira Lens, install a GitHub App named `Maira Lens` and let the
  workflow mint an installation token for it. Reviews and comments then appear
  as `maira-lens[bot]` with the app's avatar.

### Using a GitHub App

1. In organization settings, open **Developer settings → GitHub Apps → New
   GitHub App**.
2. Name it `Maira Lens`, set a homepage, and upload the logo.
3. Disable the webhook; the review workflow does not need it.
4. Grant repository permissions: **Contents: Read and write** (needed to
   resolve review threads), **Issues: Read and write**, and **Pull requests:
   Read and write**. Metadata read is implicit.
5. Install the app on the repositories that call this workflow.
6. Store the App ID as the variable `MAIRA_LENS_APP_ID` and the downloaded `.pem`
   private key as the secret `MAIRA_LENS_APP_PRIVATE_KEY`, at the organization or
   repository level. Restrict organization credentials to the repositories
   that use the reviewer. The app's client secret is only for OAuth flows and
   cannot replace the private key.
7. Pass them to the workflow:

```yaml
jobs:
  review:
    uses: gigalogy/maira-code-workflows/.github/workflows/pr-review-agent.yml@<full-sha-or-release-tag>
    permissions:
      contents: write
      issues: write
      pull-requests: write
    with:
      app_id: ${{ vars.MAIRA_LENS_APP_ID }}
      effort: high
    secrets:
      MAIRA_LENS_TOKEN: ${{ secrets.MAIRA_LENS_TOKEN }}
      MAIRA_LENS_APP_PRIVATE_KEY: ${{ secrets.MAIRA_LENS_APP_PRIVATE_KEY }}
```

If `app_id` is omitted, the workflow uses `GITHUB_TOKEN` and the author stays
`github-actions[bot]`. If `app_id` is set, `MAIRA_LENS_APP_PRIVATE_KEY` is
required and the job fails early when it is missing.

GitHub App names are globally unique, so another organization cannot create a
second app literally named `Maira Lens`. Client organizations should create
their own app, for example `Maira Lens — <client>`, with the same icon. Do not
share one app's private key across organizations.

## Repository-specific review guidance

The default brief is intentionally generic. A repository can add its own
guidance — stack, layout, domain invariants, what is critical — or replace the
default brief entirely, without maintaining a fork of the workflow.

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
      MAIRA_LENS_TOKEN: ${{ secrets.MAIRA_LENS_TOKEN }}
```

Rules and warnings:

- `prompt_ref` must be a full 40-character commit SHA; branches and tags are
  rejected. Use the base SHA, never the head or a branch: the base commit
  predates the pull request, so a pull request cannot rewrite the guidance that
  reviews it.
- `brief_mode: append` (the default) keeps the default brief and appends the
  repository brief under `## Repository-specific guidance`. The severity scale,
  untrusted-input rules, and output contract are preserved, so a short brief
  with stack and domain context is enough.
- `brief_mode: replace` uses the repository brief as the only review
  instructions. Set it only with a complete brief: the output schema still
  enforces the JSON shape, but the severity scale, confidence threshold, and
  untrusted-input rules are no longer included, so the brief must restate them.
- The brief must already exist in the base commit. Land it in a separate merged
  pull request before enabling the reviewer, otherwise the first run fails with
  `missing from <repo>@<sha>`.
- When the brief comes from the caller's own repository, `github.token` can read
  it even when private, so `MAIRA_LENS_WORKFLOWS_TOKEN` is not required.
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

## Review behavior

- Each run posts a new review and a summary comment (updated in place on later
  runs), then resolves this workflow's inline threads from the previous run.
- Inline comments are only placed on lines the diff added; everything else goes
  into the summary.
- Check names are prefixed with the caller job name, for example
  `review / Maira Lens review`.
- The caller's base commit must be available in the checkout; the workflow
  fetches it explicitly.

## Troubleshooting

| Symptom                                                | Cause and fix                                                                                                          |
|--------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| `Repository variable MAIRA_LENS_BASE_URL is not set`   | Set the variable on the calling repository or its organization.                                                        |
| `unrecognized_model`                                   | `MAIRA_LENS_MODEL` is missing or wrong; it must match the model exposed by the gateway.                                |
| `prompt_ref must be a full 40-character commit SHA`    | Use `${{ github.event.pull_request.base.sha }}`, not a branch or tag.                                                  |
| `... is missing from <repo>@<sha>`                     | The brief or prompt file does not exist at that commit. Land it first.                                                 |
| Dependency install warning                             | The stack is unsupported or the install failed; the review continues. Set `install_dependencies: false` to silence it. |
| Inline review failed, findings folded into the summary | GitHub rejected a comment position; the finding is still published in the summary.                                     |
| No review posted                                       | The PR is a draft, authored by a bot, or from a fork; the gate skipped it.                                             |
| Comments show `github-actions[bot]`                    | No `app_id` was passed, so `GITHUB_TOKEN` was used. Add the app ID and private key to post as the app.                 |
| `create-github-app-token` failed                       | The app is not installed on the repository, the app ID is wrong, or the private key does not match.                    |

## Versioning and releases

- Consumers should pin `uses:` to a full commit SHA from a release page.
- Tags in the workflow repository are the upgrade boundary. Protect release
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
