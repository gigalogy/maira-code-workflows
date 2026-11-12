# Set up Maira Code for a GitHub organization

This guide is for a GitHub organization owner or repository administrator who
wants to run Maira Code automated pull request reviews on one or more
repositories. It covers organization-level configuration, the caller workflow,
optional branding and repository-specific guidance, verification, and
troubleshooting. It can be shared with client organizations as-is.

The reviewer is published as a reusable workflow at
`gigalogy/maira-code-workflows`. Your repositories call it; nothing is
installed into your codebase.

## 1. Before you start

You need:

- Admin access to the GitHub organization, or permission to manage Actions
  settings, organization variables, and organization secrets.
- Admin access to the repositories that will use the reviewer.
- The Maira Code gateway URL, model name, and API key from Gigalogy.
- GitHub Actions enabled for the organization and the repositories.

The setup takes about 15 minutes for the first repository. Additional
repositories only need the caller workflow file.

## 2. How it fits together

```text
your repository
  .github/workflows/maira-code-review.yml   <- caller workflow (you add this)
        |
        | calls at a pinned release
        v
gigalogy/maira-code-workflows                 <- reusable workflow (public)
  .github/workflows/pr-review-agent.yml
        |
        | reads organization/repository configuration
        v
MAIRA_CODE_BASE_URL + MAIRA_CODE_MODEL (variables)
MAIRA_CODE_TOKEN (secret)
        |
        | reviews the PR and posts feedback
        v
pull request: inline comments + summary, posted as github-actions[bot]
              or as a GitHub App if you configure one
```

## 3. Configure organization-level access

### 3.1 Actions policy

Open **Organization settings → Actions → General**.

- Under **Actions permissions**, allow GitHub Actions. If your organization
  restricts actions, use **Allow select actions and reusable workflows** and
  add:

  ```text
  gigalogy/maira-code-workflows/.github/workflows/pr-review-agent.yml@*
  actions/checkout@*
  actions/setup-python@*
  actions/github-script@*
  astral-sh/setup-uv@*
  ```

  Add `actions/create-github-app-token@*` only if you enable the optional
  GitHub App branding in section 5. `astral-sh/setup-uv` is only used for
  Python projects that use `uv.lock`.

- Under **Workflow permissions**, select **Read and write permissions**. The
  reviewer posts comments and resolves review threads. The caller workflow's
  own `permissions:` block is capped by this organization setting, so a
  read-only policy makes the review fail even if the caller asks for write
  access.

> GitHub Enterprise Server requires the reusable workflow to be reachable from
> your server. The public `github.com` workflow path in this guide does not
> apply to GHES without a mirror.

### 3.2 Organization variables and secrets

Open **Organization settings → Secrets and variables → Actions**.

Create these **variables** (visible configuration, not secret):

| Variable              | Example                     | Purpose                                                        |
|-----------------------|-----------------------------|----------------------------------------------------------------|
| `MAIRA_CODE_BASE_URL` | `https://maira.example.com` | Gateway API root. Do not append `/v1`; the CLI appends it.     |
| `MAIRA_CODE_MODEL`    | `g-maira-code`              | Model exposed by the gateway.                                  |
| `REVIEW_SHOW_COST`    | `false`                     | Optional. Set to `false` to hide the cost line in summaries.   |

Create this **secret**:

| Secret             | Purpose                                             |
|--------------------|-----------------------------------------------------|
| `MAIRA_CODE_TOKEN` | Maira Hub API key, sent as the `X-Api-Key` header. |

Under **Repository access**, choose the repositories that will run the
reviewer, or select all repositories. Secrets are not readable from workflow
files or logs; variables are visible to organization members.

Never put the API key in a workflow file, a repository variable, or a commit.

### 3.3 Optional `WORKFLOWS_TOKEN`

You do not need this for the public workflow repository or for a
repository-specific brief stored in the caller's own repository. Create it only
when `prompt_repository` points at a separate **private** repository; the token
needs `contents: read` on that repository and is passed as `WORKFLOWS_TOKEN`.

## 4. Add the caller workflow

Create `.github/workflows/maira-code-review.yml` in each repository:

```yaml
name: Maira Code review

on:
  pull_request:
    branches: [main]          # or your default/review branches
    types: [opened, reopened, synchronize, ready_for_review, review_requested]

concurrency:
  group: maira-code-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true

jobs:
  review:
    uses: gigalogy/maira-code-workflows/.github/workflows/pr-review-agent.yml@<full-sha-or-release-tag>
    permissions:
      contents: write        # the thread-resolution job needs this
      issues: write
      pull-requests: write
    with:
      effort: high
      max_budget_usd: "10"
    secrets:
      MAIRA_CODE_TOKEN: ${{ secrets.MAIRA_CODE_TOKEN }}
```

Replace `<full-sha-or-release-tag>` with a release tag or, preferably, the full
commit SHA shown on the release page. Never pin to a branch.

What each part does:

- **`on.pull_request`** — the reviewer only runs for pull request events. Do
  not switch this to `pull_request_target`; the workflow's safety gate expects
  `pull_request`.
- **`concurrency`** — cancels older runs when the PR is updated.
- **`permissions`** — the caller is the ceiling. The review job itself runs
  with `contents: read`; resolving previous review threads happens in a
  separate job that needs `contents: write`.
- **`with.effort`** — `low`, `medium`, `high`, or `max`. Higher effort costs
  more and takes longer.
- **`with.max_budget_usd`** — spending ceiling for one pass, shared across
  retry attempts.
- **`secrets.MAIRA_CODE_TOKEN`** — the organization secret from section 3.2.

The workflow only runs for **non-draft, human-authored pull requests from
branches in the same repository**. Fork pull requests are skipped so they never
receive the gateway key.

### Common inputs

| Input                  | Default                        | When to change it                                                        |
|------------------------|--------------------------------|--------------------------------------------------------------------------|
| `effort`               | `high`                         | Lower it to reduce cost; raise it for security-sensitive repositories.   |
| `max_budget_usd`       | `"10"`                         | Lower per-repository budgets for high-volume repos.                      |
| `project_directory`    | `"."`                          | Set it when the app lives in a subdirectory (monorepos).                 |
| `install_dependencies` | `true`                         | Set `false` for unsupported stacks or to skip environment setup.         |
| `app_id`               | `""`                           | Set it to post as a GitHub App instead of `github-actions[bot]`.          |
| `prompt_repository`    | `""`                           | Repository containing an additional review brief.                        |
| `prompt_ref`           | `""`                           | Full 40-character base SHA for `prompt_repository`.                      |
| `prompt_path`          | `.github/maira/pr-review.md`   | Location of the additional brief inside `prompt_repository`.             |
| `brief_mode`           | `append`                       | `append` keeps the default brief; `replace` uses only your brief.        |

## 5. Optional: brand the reviewer

By default the workflow posts with the repository's `GITHUB_TOKEN`, which
GitHub always displays as **`github-actions[bot]`**. The review text already
says Maira Code, but the author badge cannot be renamed.

To post as **Maira Lens** with your logo:

1. Open **Organization settings → Developer settings → GitHub Apps → New
   GitHub App**.
2. Name the app `Maira Lens`, set a homepage, and upload the logo. The slug
   becomes `maira-lens`, so comments appear from `maira-lens[bot]`.
3. Disable the webhook; the reviewer does not need it.
4. Grant repository permissions:
   - **Contents: Read and write** (required to resolve review threads)
   - **Issues: Read and write**
   - **Pull requests: Read and write**
   - Metadata read is implicit.
5. Install the app on the repositories that call the reviewer.
6. Store the credentials at the **organization** level (recommended when more
   than one repository uses the reviewer) or at the repository level:

   - **Variable** `MAIRA_LENS_APP_ID`: the app's **App ID**. Do not use the Client
     ID or the client secret here.
   - **Secret** `MAIRA_LENS_APP_PRIVATE_KEY`: the `.pem` private key generated on
     the app's settings page.

   If you create organization credentials, restrict their repository access to
   the repositories that call the reviewer.

   GitHub Apps authenticate server-to-server with the App ID and a private key
   to mint installation tokens. The **client secret** is only used for OAuth
   authorization-code flows and cannot substitute for the private key;
   `actions/create-github-app-token` requires `app-id` and `private-key`. Treat
   the private key as a long-lived credential: store it only as a secret,
   restrict its repository access, and rotate it if it is ever exposed.
7. Pass them to the workflow:

```yaml
    with:
      app_id: ${{ vars.MAIRA_LENS_APP_ID }}
      effort: high
    secrets:
      MAIRA_CODE_TOKEN: ${{ secrets.MAIRA_CODE_TOKEN }}
      app_private_key: ${{ secrets.MAIRA_LENS_APP_PRIVATE_KEY }}
```

The app token is minted only after the model has finished reviewing, so the
private key is not present while the agent processes untrusted PR content. If
`app_id` is set but the private key is missing, the job fails early with a
clear error.

App branding requires a release that includes the `app_id` input. Check the
release notes before pinning; `v1.0.0` does not include it.

GitHub App names are globally unique. If Gigalogy already owns an app named
`Maira Lens`, another organization cannot create a second one with the same
name. Use a name such as `Maira Lens — <company>` with the same icon. Do not
share one app's private key across organizations.

## 6. Optional: repository-specific review guidance

The default brief is deliberately generic and works across languages and
frameworks. Add a repository brief when you want the reviewer to know your
stack, layout, and domain invariants.

Create `.github/maira/pr-review.md` in the repository:

```markdown
### Stack

Laravel 11 API with MySQL, Redis queues, and a Vue 3 admin panel.

### Layout

- `app/` domain code; `app/Http/Controllers` stays thin.
- `database/migrations` is append-only; never edit a released migration.

### Invariants that matter

- Every tenant-owned query must go through `TenantScope`; a missing scope is a
  critical finding.
- Jobs must be idempotent: `ShouldBeUnique` or an explicit dedupe key.
- Money is stored as integer minor units; never introduce floats.

### Out of scope

- `resources/js/components/ui/` is vendored from the design system.
- Generated files under `bootstrap/cache/`.
```

Enable it in the caller workflow:

```yaml
    with:
      prompt_repository: ${{ github.repository }}
      prompt_ref: ${{ github.event.pull_request.base.sha }}
      prompt_path: .github/maira/pr-review.md
      brief_mode: append
```

Rules:

- `prompt_ref` must be a **full 40-character commit SHA**. Branches and tags
  are rejected. Use the PR's **base** SHA, never the head: the base commit
  predates the pull request, so a PR cannot rewrite the guidance that reviews
  it.
- `brief_mode: append` (default) adds your brief beneath the default brief, so
  the severity scale, untrusted-input handling, and output contract stay
  intact.
- `brief_mode: replace` uses only your brief. Use it only with a complete brief
  that restates the severity scale, the 0.7 confidence threshold, and the
  untrusted-input rules; the JSON schema still enforces the output shape.
- Land the brief in a merged PR **before** enabling the reviewer. The first run
  reads it from the base commit and fails if the file is not there yet.
- When the brief lives in the caller's own repository, `github.token` reads it
  even when the repository is private, so `WORKFLOWS_TOKEN` is not needed.

## 7. Dependencies and monorepos

Before the review starts, the workflow installs the project environment when it
recognizes the stack:

| Stack                 | Detected by                                   | Install                                  |
|-----------------------|-----------------------------------------------|------------------------------------------|
| Python                | `uv.lock`, `poetry.lock`, `requirements.txt`, `pyproject.toml`, `setup.py` | `uv sync`, `poetry install`, `pip install` |
| Node.js / TypeScript  | `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `bun.lockb` | `npm ci`, `yarn`, `pnpm`, `bun`          |
| PHP                   | `composer.json`                               | `composer install`                       |
| Java / Kotlin         | `pom.xml`, `gradlew`, `build.gradle(.kts)`    | `mvn dependency:go-offline`, `gradle dependencies` |
| Go                    | `go.mod`                                      | `go mod download`                        |
| Rust                  | `Cargo.toml`                                  | `cargo fetch`                            |
| Ruby                  | `Gemfile`                                     | `bundle install`                         |
| .NET                  | `*.csproj`, `*.sln`                           | `dotnet restore`                         |

- Install failures are warnings; the review continues from source.
- Set `project_directory` when the app lives in a subdirectory. The diff is
  always taken across the whole repository.
- Set `install_dependencies: false` if your stack is not listed or you do not
  want the workflow to run package managers.

## 8. Verify the setup

1. Create a branch in the repository (not a fork) and open a non-draft pull
   request.
2. Open the **Actions** tab. The caller workflow appears as
   **Maira Code review**, and the job appears as **Maira Code review**.
3. The check name is prefixed with the caller job id, for example
   `review / Maira Code review`.
4. After the first run, the PR has a summary comment titled
   **Maira Code review** and, where GitHub allows, inline comments on added
   lines.
5. Push another commit. The next run updates the summary in place and resolves
   the previous run's inline threads.

Draft PRs, bot-authored PRs, and fork PRs are intentionally skipped.

## 9. Security and operations

- **Fork safety** — the workflow only runs for same-repository PRs, so fork PRs
  never receive the gateway key.
- **Least privilege** — the review job runs with `contents: read` and an agent
  command allowlist that blocks file edits, commits, pushes, `curl`, `wget`,
  package installs, and `gh`. Thread resolution runs in a separate job with no
  checkout and no untrusted-content processing.
- **Untrusted input** — PR titles, descriptions, diffs, comments, and file
  contents are treated as untrusted; the agent is instructed to report prompt
  injection rather than follow it.
- **Build scripts** — dependency installation and allowed test commands run
  repository code with the gateway key present in the job environment. Only
  collaborators who can push branches can reach that path; use protected
  branches so outside contributors cannot create in-repository branches.
- **Secrets** — rotate `MAIRA_CODE_TOKEN` and the app private key on your normal
  schedule. The gateway key is scrubbed from published output and masked in
  logs.
- **Cost** — `max_budget_usd` caps each pass and is shared across retries. Set
  `REVIEW_SHOW_COST=false` if you do not want the cost shown in PR summaries.
- **Workflow pinning** — pin the reusable workflow to a release SHA. Update it
  deliberately, the same way you would update any dependency.

## 10. Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `Repository variable MAIRA_CODE_BASE_URL is not set` | The organization variable is missing, or the repository is not in its access list. |
| `unrecognized_model` | `MAIRA_CODE_MODEL` is missing or does not match the gateway's model name. |
| `Repository secret MAIRA_CODE_TOKEN is not set` | The secret is missing or the repository is not in its access list. |
| `prompt_ref must be a full 40-character commit SHA` | Use `${{ github.event.pull_request.base.sha }}`, not a branch or tag. |
| `... is missing from <repo>@<sha>` | The brief does not exist in the base commit. Merge it first. |
| Dependency install warning | The stack is unsupported or the install failed. The review continues; set `install_dependencies: false` to silence it. |
| No review posted | The PR is a draft, authored by a bot, or from a fork, so the safety gate skipped it. |
| Workflow fails with a permissions error | Organization workflow permissions are read-only. Set **Read and write permissions**, and keep the caller `permissions:` block. |
| `create-github-app-token` failed | The app is not installed on the repository, the app ID is wrong, or the private key does not match. |
| Comments still show `github-actions[bot]` | `app_id` was not passed, so `GITHUB_TOKEN` was used. See section 5. |
| Inline review failed, findings folded into the summary | GitHub rejected a comment position; the findings are still published in the summary. |

## 11. Rollout checklist

- [ ] Actions policy allows the reusable workflow and the actions it uses.
- [ ] Organization workflow permissions are read and write.
- [ ] `MAIRA_CODE_BASE_URL` and `MAIRA_CODE_MODEL` variables exist and are
      accessible to the target repositories.
- [ ] `MAIRA_CODE_TOKEN` secret exists and is accessible to the target
      repositories.
- [ ] Caller workflow added to a pilot repository and pinned to a release SHA.
- [ ] Test PR reviewed successfully; summary and check name look correct.
- [ ] Optional: GitHub App installed and `app_id`/`app_private_key` configured.
- [ ] Optional: repository brief merged and `prompt_*` inputs configured.
- [ ] `project_directory` and `install_dependencies` reviewed for monorepos and
      unsupported stacks.
- [ ] Rolled out to the remaining repositories.

## Appendix: configuration reference

| Name | Kind | Scope | Required | Purpose |
|---|---|---|---|---|
| `MAIRA_CODE_BASE_URL` | variable | organization or repository | yes | Gateway API root, without `/v1`. |
| `MAIRA_CODE_MODEL` | variable | organization or repository | yes | Model exposed by the gateway. |
| `MAIRA_CODE_TOKEN` | secret | organization or repository | yes | Gateway API key, sent as `X-Api-Key`. |
| `REVIEW_SHOW_COST` | variable | organization or repository | no | `false` hides the cost line. |
| `MAIRA_LENS_APP_ID` | variable | organization or repository | no | GitHub App ID for branded comments. |
| `MAIRA_LENS_APP_PRIVATE_KEY` | secret | organization or repository | no | GitHub App private key. |
| `WORKFLOWS_TOKEN` | secret | organization or repository | no | Read access to a private prompt repository. |
