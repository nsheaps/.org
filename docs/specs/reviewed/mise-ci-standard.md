# Mise & CI Workflow Standard for nsheaps Repos

**Status**: Reviewed
**Date**: 2026-02-23
**Author**: Foghorn L (ops-eng)
**Baseline**: [homebrew-devsetup](https://github.com/nsheaps/homebrew-devsetup) (most mature CI), [claude-utils](https://github.com/nsheaps/claude-utils) (release pattern)

## Problem

nsheaps repos have inconsistent mise configuration and CI workflows:

- Task names vary (`fmt` vs `format`, `fmt-check` vs `format-check`)
- Some repos have no tasks defined at all
- CI workflow structure differs (some have auto-format, some don't)
- No shared standard for what "check" or "release" means
- No consistent dependency update policy

This makes it harder to switch between repos, onboard contributors, and maintain CI.

## Scope

This standard applies to all nsheaps repos that have code to format, lint, or test. Minimal repos (like `.org`) that contain only documentation or configuration may be exempt.

## Standard

### 1. Filename: `.mise.toml`

Use `.mise.toml` at the repo root. Prefer the dot-prefixed filename.

- Rationale: Consistent with `.mise/tasks/` directory convention; keeps configuration files hidden alongside their task scripts
- Reference: [mise docs — Configuration](https://mise.jdx.dev/configuration.html)

**Repos requiring rename**: claude-utils, homebrew-devsetup, agent-team, op-exec, git-wt, github-actions (currently use `mise.toml` without dot prefix)

### 2. Task Names

All repos MUST define these standard task names where applicable:

| Task       | Description                                                    | Required?                                                                                                                               |
| :--------- | :------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------- |
| `fix`      | Auto-fix code: formatters (prettier, gofmt) + lint fix modes   | Always                                                                                                                                  |
| `lint`     | Run all linters in read-only mode (no modifications)           | Always                                                                                                                                  |
| `test`     | Run test suite                                                 | Always. Exits 0 if no tests exist AND repo has < 10k LOC. Otherwise fails with message that tests must be defined or the task removed. |
| `security` | Run security scanning tools (see section 7)                    | Recommended                                                                                                                             |
| `check`    | CI meta-task: depends on fix, test, and security               | Always                                                                                                                                  |

Rules:

- Use `fix`, NOT `format` or `fmt`
- `fix` runs formatters AND lint auto-fix modes. If `fix` passes, `lint` should also pass.
- `lint` is a **local-only convenience task** — it runs read-only checks without modifying files. It is NOT part of the CI `check` chain because `fix` already covers everything `lint` would catch, plus applies fixes. Use `lint` locally to preview what would fail without auto-fixing.
- `check` is the CI meta-task — it depends on `fix`, `test`, and `security`
- Additional repo-specific tasks (e.g., `lint-formula` for Homebrew repos) are fine alongside the standard names

### 3. Task Definition Style

Always use **file-based `.mise/tasks/`** for task definitions. Do not use inline `[tasks.*]` in `.mise.toml`.

- Rationale: File-based tasks can be run without mise as standalone scripts, are easier to lint and test, and provide better discoverability
- Reference: [mise docs — Tasks](https://mise.jdx.dev/tasks/)

Example file-based task (`.mise/tasks/fix`):

```bash
#!/usr/bin/env bash
#MISE description="Auto-fix: format + lint fix modes"
set -euo pipefail

bun run prettier --write '**/*.md'
```

Example file-based task (`.mise/tasks/test`):

```bash
#!/usr/bin/env bash
#MISE description="Run test suite"
set -euo pipefail

# If no tests exist and repo is small, exit 0
if [ ! -d "tests" ] && [ ! -d "test" ] && [ ! -d "__tests__" ]; then
  loc=$(find . -name '*.ts' -o -name '*.js' -o -name '*.go' -o -name '*.sh' | xargs wc -l 2>/dev/null | tail -1 | awk '{print $1}')
  if [ "${loc:-0}" -lt 10000 ]; then
    echo "No tests found and repo has < 10k LOC. Passing."
    exit 0
  else
    echo "::error::Repo has >= 10k LOC but no tests defined. Add tests or remove the test task."
    exit 1
  fi
fi

bun test
```

Example file-based task (`.mise/tasks/security`):

```bash
#!/usr/bin/env bash
#MISE description="Run security scanning tools"
set -euo pipefail

# Run whichever tools are installed — see section 7
command -v gitleaks &>/dev/null && gitleaks detect --no-banner || true
command -v trufflehog &>/dev/null && trufflehog filesystem . --no-update || true
echo "Security scan complete."
```

Example file-based task (`.mise/tasks/check`):

```bash
#!/usr/bin/env bash
#MISE description="Run all checks (fix, test, security)"
#MISE depends=["fix", "test", "security"]
set -euo pipefail

echo "All checks passed."
```

### 4. CI Workflow: `check.yaml`

Every repo MUST have `.github/workflows/check.yaml` that runs on PRs and pushes to main.

#### Triggers

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
```

#### Concurrency

```yaml
concurrency:
  group: >-
    ${{ github.event_name == 'push' && github.sha ||
        format('{0}-{1}', github.workflow, github.ref) }}
  cancel-in-progress: true
```

#### Fix Job: Auto-Fix + Commit + Fail

This is the core pattern from homebrew-devsetup. The fix job:

1. Checks out the code with a GitHub App token (for push access)
2. Installs tools via mise
3. Runs `mise run fix`
4. Checks for changes via `git status --porcelain`
5. If changes exist: auto-commits them and exits with failure (triggers re-run)
6. If no changes: passes

```yaml
env:
  GH_APP_ID: ${{ secrets.GH_APP_ID }}
  GH_APP_PRIVATE_KEY: ${{ secrets.GH_APP_PRIVATE_KEY }}

jobs:
  fix:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/github-app-auth
      - name: Check for merge conflicts
        run: |
          if git diff --check HEAD; then
            echo "No merge conflict markers found."
          else
            echo "::error::Merge conflict markers detected"
            exit 1
          fi
      - uses: jdx/mise-action@v3
      - name: Fix
        run: mise run fix
      - name: Check for changes
        id: changes
        run: |
          if [ -n "$(git status --porcelain)" ]; then
            echo "has_changes=true" >> "$GITHUB_OUTPUT"
          fi
      - name: Commit fix changes
        if: steps.changes.outputs.has_changes == 'true'
        uses: stefanzweifel/git-auto-commit-action@v7
        with:
          commit_message: "style: auto-fix"
      - name: Fail if changes were made
        if: steps.changes.outputs.has_changes == 'true'
        run: |
          echo "::error::Files were auto-fixed and committed. CI will re-run."
          exit 1
```

Key dependencies:

- **`stefanzweifel/git-auto-commit-action@v7`** for auto-committing fix changes
- **`jdx/mise-action@v3`** for installing mise and tools
- **`.github/actions/github-app-auth`** composite action (see section 6)

#### Check Job

Runs after the fix job passes. Calls `mise run check` (which depends on `fix`, `test`, and `security`).

> **Note on intentional redundancy**: `mise run check` depends on `fix`, which means `fix` runs twice in CI — once in the standalone fix job (which auto-commits changes) and once as part of `check`. This is intentional. The fix job exists to auto-commit formatting changes and fail the run. The `check` task's dependency on `fix` ensures `mise run check` is self-contained and correct when run locally. The second `fix` run in CI is a no-op if the first pass already committed everything.

```yaml
  check:
    runs-on: ubuntu-latest
    needs: [fix]
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: jdx/mise-action@v3
      - name: Check
        run: mise run check
```

### 5. CI Workflow: `release.yaml`

Repos that publish releases SHOULD have `.github/workflows/release.yaml`. Use [release-it](https://github.com/release-it/release-it) when a release tool is needed, but do not standardize on a specific release tool — each repo may use what fits best.

### 6. GitHub App Auth Composite Action

Every repo using auto-fix or release workflows MUST include `.github/actions/github-app-auth/action.yml`.

This composite action:

1. Generates a GitHub App token via `actions/create-github-app-token@v2`
2. Gets the bot user ID from the GitHub API
3. Configures git user name and email for the bot
4. Re-checkouts the repo with the app token (for push access)

```yaml
name: GitHub App Auth
description: Authenticate as a GitHub App for CI operations
outputs:
  token:
    description: GitHub App token
    value: ${{ steps.app-token.outputs.token }}
  app-slug:
    description: GitHub App slug
    value: ${{ steps.app-token.outputs.app-slug }}
runs:
  using: composite
  steps:
    - name: Generate token
      id: app-token
      uses: actions/create-github-app-token@v2
      with:
        app-id: ${{ env.GH_APP_ID }}
        private-key: ${{ env.GH_APP_PRIVATE_KEY }}
    - name: Get bot user ID
      id: bot-user
      shell: bash
      run: |
        echo "user-id=$(gh api "/users/${{ steps.app-token.outputs.app-slug }}[bot]" --jq .id)" >> "$GITHUB_OUTPUT"
        echo "user-name=${{ steps.app-token.outputs.app-slug }}[bot]" >> "$GITHUB_OUTPUT"
      env:
        GH_TOKEN: ${{ steps.app-token.outputs.token }}
    - name: Configure git
      shell: bash
      run: |
        git config user.name "${{ steps.bot-user.outputs.user-name }}"
        git config user.email "${{ steps.bot-user.outputs.user-id }}+${{ steps.bot-user.outputs.user-name }}@users.noreply.github.com"
    - name: Checkout with token
      uses: actions/checkout@v4
      with:
        token: ${{ steps.app-token.outputs.token }}
```

Required secrets (set org-wide or per-repo):

- `GH_APP_ID` — GitHub App ID
- `GH_APP_PRIVATE_KEY` — GitHub App private key

### 7. Security Scanning

Security scanning is recommended for all repos. Based on homebrew-devsetup:

Tools (installed via mise):

- `gitleaks` — git secrets detection
- `trufflehog` — credential scanning
- `grype` — vulnerability scanning
- `trivy` — container/filesystem scanning
- `checkov` — infrastructure-as-code scanning
- `secretlint` — secret pattern matching
- `syft` — SBOM generation
- KICS — via GitHub Action (not mise)

These run in parallel via `qoomon/actions--parallel-steps@v1`.

### 8. Dependency Management: Renovate

All repos MUST use [nsheaps/renovate-config](https://github.com/nsheaps/renovate-config) for automated dependency updates.

- Each repo should have a `renovate.json` (or `renovate.json5`) that extends `nsheaps/renovate-config`
- Renovate handles dependency PRs for npm packages, GitHub Actions versions, Docker images, and tool versions

### 9. Org Compliance

A workflow in `nsheaps/.org` SHOULD use the automation GitHub App to check repos in the org for compliance with this standard. This enables ongoing enforcement rather than one-time rollout.

## Rollout Phases

Phases MUST be executed in order. Phase 1 should land before Phase 2 to avoid merge conflicts on renamed files.

- **Phase 1** — Filename renames (`mise.toml` -> `.mise.toml` for repos that use the non-dot-prefixed name)
- **Phase 2** — Task standardization (add/rename standard task names, move to file-based `.mise/tasks/`)
- **Phase 3** — CI workflow standardization (rename `test.yaml` -> `check.yaml`, add auto-fix+commit+fail pattern, add `github-app-auth` composite action)

## Changes Required Per Repo

| Repo              | Rename File                   | Task Changes                                   | CI Changes                                                             | Notes                            |
| :---------------- | :---------------------------- | :--------------------------------------------- | :--------------------------------------------------------------------- | :------------------------------- |
| ai-mktpl          | Already `.mise.toml`          | Add fix, lint, test, check as file-based tasks | Create `check.yaml` with auto-fix; retire existing CI if redundant     | Has justfile — leave as-is       |
| .github           | Already `.mise.toml`          | Add fix, lint, test, check as file-based tasks | Rename `ci.yaml` -> `check.yaml`, add auto-fix                        | Has many custom tasks; keep them |
| aitkit            | Already `.mise.toml`          | Add fix, lint, test, check as file-based tasks | Already has auto-format; rename `lint.yml`+`test.yml` -> `check.yaml` | Go project                       |
| agent-team        | `mise.toml` -> `.mise.toml`   | Rename fmt->fix; add file-based tasks          | Rename `test.yaml` -> `check.yaml`, add auto-fix                      | PR #97 needs update              |
| claude-utils      | `mise.toml` -> `.mise.toml`   | Add fix, lint, test, check as file-based tasks | Rename `test.yaml` -> `check.yaml`, add auto-fix                      |                                  |
| op-exec           | `mise.toml` -> `.mise.toml`   | Add fix, lint, test, check as file-based tasks | Rename `test.yaml` -> `check.yaml`, add auto-fix                      |                                  |
| git-wt            | `mise.toml` -> `.mise.toml`   | Add fix, lint, test, check as file-based tasks | Rename `test.yaml` -> `check.yaml`, add auto-fix                      |                                  |
| homebrew-devsetup | `mise.toml` -> `.mise.toml`   | Add fix task; rename existing tasks             | Already has `check.yaml` with auto-format                              | Gold standard for CI pattern     |
| github-actions    | `mise.toml` -> `.mise.toml`   | Verify/add standard task names as file-based   | Already has `check.yaml`                                               | Verify alignment                 |
| renovate-config   | Already `.mise.toml`          | Add fix, lint, test, check as file-based tasks | Rename `check.yml` -> `check.yaml`, verify pattern                     | Uses `.yml` extension            |
| .org              | N/A                           | N/A                                            | Compliance workflow (see section 9)                                    | Exempt from standard tasks       |

## Existing Draft PRs

These PRs were created ahead of this spec and need revision after approval:

**Phase 1 — Filename renames:**

The following PRs renamed in the WRONG direction (`.mise.toml` -> `mise.toml`). They should be **closed** since the standard now prefers `.mise.toml`:

- [ai-mktpl#189](https://github.com/nsheaps/ai-mktpl/pull/189) — **CLOSE**. Verified: 6 files changed, all rename-related (`.mise.toml` → `mise.toml` rename + 5 reference updates in docs/scripts/hooks). No functional changes, but direction is now wrong.
- [.github#18](https://github.com/nsheaps/.github/pull/18) — **CLOSE** (already uses `.mise.toml`)
- [aitkit#12](https://github.com/nsheaps/aitkit/pull/12) — **CLOSE** (already uses `.mise.toml`)

New Phase 1 PRs needed for repos currently using `mise.toml` (without dot):

- agent-team, claude-utils, op-exec, git-wt, homebrew-devsetup, github-actions

**Phase 2 — Task standardization:**

These PRs need updating to use `fix` (not `format`) and file-based tasks (not inline):

- [agent-team#97](https://github.com/nsheaps/agent-team/pull/97) — **needs update**: currently renames `fmt`→`format` (should be `fix`), uses inline `[tasks.*]` (should be file-based `.mise/tasks/`), and modifies `test.yaml` (should be renamed to `check.yaml`)
- [claude-utils#6](https://github.com/nsheaps/claude-utils/pull/6) — **needs update**: uses `format`, inline tasks
- [op-exec#1](https://github.com/nsheaps/op-exec/pull/1) — **needs update**: uses `format`, inline tasks
- [git-wt#10](https://github.com/nsheaps/git-wt/pull/10) — **needs update**: uses `format`, inline tasks
- [homebrew-devsetup#103](https://github.com/nsheaps/homebrew-devsetup/pull/103) — **needs update**: uses `format`, inline tasks

**Phase 3 — CI workflow standardization (not yet started):**

To be created after spec approval:

- Rename `test.yaml` -> `check.yaml` in agent-team, claude-utils, op-exec, git-wt
- Add auto-fix+commit+fail pattern to all `check.yaml` workflows
- Add `.github/actions/github-app-auth/action.yml` composite action to repos that don't have it
- Map secrets to env vars (`GH_APP_ID`, `GH_APP_PRIVATE_KEY`) at workflow level

## References

- [mise Documentation](https://mise.jdx.dev/)
- [mise Configuration](https://mise.jdx.dev/configuration.html)
- [mise Tasks](https://mise.jdx.dev/tasks/)
- [stefanzweifel/git-auto-commit-action](https://github.com/stefanzweifel/git-auto-commit-action)
- [actions/create-github-app-token](https://github.com/actions/create-github-app-token)
- [jdx/mise-action](https://github.com/jdx/mise-action)
- [Conventional Commits](https://www.conventionalcommits.org/)
- [release-it Documentation](https://github.com/release-it/release-it)
- [nsheaps/renovate-config](https://github.com/nsheaps/renovate-config)
- Audit report: `nsheaps/agent-team:.claude/tmp/mise-consistency-audit.md`
- Draft proposal: `nsheaps/agent-team:.claude/tmp/mise-standard-proposal.md`
