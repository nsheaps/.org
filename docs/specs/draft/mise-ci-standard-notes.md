# Mise & CI Workflow Standard for nsheaps Repos

**Status**: Draft
**Date**: 2026-02-23
**Author**: Foghorn L (ops-eng)
**Baseline**: [homebrew-devsetup](https://github.com/nsheaps/homebrew-devsetup) (most mature CI), [claude-utils](https://github.com/nsheaps/claude-utils) (release pattern)

## Problem

nsheaps repos have inconsistent mise configuration and CI workflows:

- Mixed filenames (`.mise.toml` vs `mise.toml`)
- Task names vary (`fmt` vs `format`, no standard `fix` task)
- Some repos have no tasks defined at all; others use inline definitions
- CI workflow structure differs (some have auto-fix, some don't)
- No shared standard for what "check" or "release" means

This makes it harder to switch between repos, onboard contributors, and maintain CI.

## Scope

This standard applies to all nsheaps repos that have code to format, lint, or test. Minimal repos (like `.org`) that contain only documentation or configuration may be exempt.

A workflow in nsheaps/.org SHOULD use the automation GitHub App to check repos in the org for compliance with this standard.

## Standard

### 1. Filename: `.mise.toml`

Use `.mise.toml` (dot-prefixed) at the repo root, with tasks in `.mise/tasks/`.

- Rationale: Consistent with `.mise/tasks/` directory convention; keeps tooling config alongside other dotfiles
- Reference: [mise docs — Configuration](https://mise.jdx.dev/configuration.html)

**Repos requiring rename**: agent-team, claude-utils, op-exec, git-wt, homebrew-devsetup, github-actions (currently use `mise.toml` without dot prefix)

### 2. Task Names

All repos MUST define these standard task names where applicable:

| Task    | Description                                                          | Required?                                                                                                                        |
| :------ | :------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------- |
| `fix`   | Auto-fix everything: format + lint --fix where available             | Always                                                                                                                           |
| `lint`  | Run all linters (check-only, no modifications)                       | Always                                                                                                                           |
| `test`  | Run test suite                                                       | Always — exits 0 if no tests exist AND repo has < 10k LOC; otherwise fails with message that tests must be added or job removed |
| `check` | Run fix + test + security (the CI meta-task)                         | Always                                                                                                                           |

Rules:

- Use `fix`, NOT `format` — `fix` runs formatters (prettier, gofmt) AND lint auto-fix modes. If `fix` passes, `lint` should also pass.
- `check` is the CI meta-task that runs everything — it should depend on `fix`, `test`, and optionally `security`
- `lint` is check-only (no writes). `fix` is the write-mode counterpart.
- Additional repo-specific tasks (e.g., `lint-formula` for Homebrew repos) are fine alongside the standard names
- Repos with existing `justfile` configurations: leave justfiles as-is; don't change them as part of this standardization

### 3. Task Definition Style

Always use **file-based `.mise/tasks/`** for task definitions.

- Rationale: Scripts in `.mise/tasks/` can be run directly without mise installed, improving portability
- Each task is a standalone script with a shebang (e.g., `#!/usr/bin/env bash`)
- Reference: [mise docs — File Tasks](https://mise.jdx.dev/tasks/file-tasks.html)

Example `.mise/tasks/fix`:

```bash
#!/usr/bin/env bash
# mise description="Auto-fix: format + lint --fix"
set -euo pipefail

bun run prettier --write '**/*.md'
```

Example `.mise/tasks/lint`:

```bash
#!/usr/bin/env bash
# mise description="Run all linters (check-only)"
set -euo pipefail

bun run prettier --check '**/*.md'
```

Example `.mise/tasks/test`:

```bash
#!/usr/bin/env bash
# mise description="Run test suite"
set -euo pipefail

# Repos under 10k LOC without tests may exit 0
# Larger repos MUST have tests or remove this job
bun test
```

Example `.mise/tasks/check`:

```bash
#!/usr/bin/env bash
# mise description="Run all checks (fix + test)"
# mise depends=["fix", "test"]
set -euo pipefail

echo "All checks passed"
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
      - name: Commit fixes
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

- **`stefanzweifel/git-auto-commit-action@v7`** for auto-committing fixes
- **`jdx/mise-action@v3`** for installing mise and tools
- **`.github/actions/github-app-auth`** composite action (see section 6)
- **Renovate** using [nsheaps/renovate-config](https://github.com/nsheaps/renovate-config) for dependency updates

#### Test Job

Runs after fix passes.

```yaml
  test:
    runs-on: ubuntu-latest
    needs: [fix]
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: jdx/mise-action@v3
      - name: Test
        run: mise run test
```

#### Security Job

Runs in parallel with test. See section 7 for details.

### 5. CI Workflow: `release.yaml`

Repos that publish releases MUST have `.github/workflows/release.yaml`. Use release-it when needed; do not standardize on a specific release tool for now.

Based on claude-utils pattern:

```yaml
name: Release
on:
  push:
    branches: [main]
concurrency:
  group: release
  cancel-in-progress: false
env:
  GH_APP_ID: ${{ secrets.GH_APP_ID }}
  GH_APP_PRIVATE_KEY: ${{ secrets.GH_APP_PRIVATE_KEY }}
jobs:
  release:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: ./.github/actions/github-app-auth
      - uses: jdx/mise-action@v3
      - name: Install dependencies
        run: bun install --frozen-lockfile  # or: yarn install --frozen-lockfile
      - name: Release
        run: bun run release-it --ci  # or: yarn release-it --ci
        env:
          GITHUB_TOKEN: ${{ steps.github-app-auth.outputs.token }}
```

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

Repos SHOULD include a security job in `check.yaml`. Based on homebrew-devsetup:

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

Recommended for all repos, especially those with:

- Published binaries or packages
- Infrastructure-as-code
- Secrets management
- Public-facing tooling

### 8. Renovate

All repos MUST use [nsheaps/renovate-config](https://github.com/nsheaps/renovate-config) for automated dependency updates.

Add a `renovate.json5` at the repo root:

```json5
{
  $schema: "https://docs.renovatebot.com/renovate-schema.json",
  extends: ["github>nsheaps/renovate-config"],
}
```

## Rollout Phases

Phases MUST be executed in order. Phase 1 should land before Phase 2 to avoid merge conflicts on renamed files.

- **Phase 1** — Filename renames (`mise.toml` -> `.mise.toml`); move inline tasks to `.mise/tasks/`
- **Phase 2** — Task standardization (add/rename to `fix`, `lint`, `test`, `check`)
- **Phase 3** — CI workflow standardization (rename `test.yaml` -> `check.yaml`, add auto-fix+commit+fail pattern, add `github-app-auth` composite action, add Renovate config)

## Changes Required Per Repo

| Repo              | Rename File                         | Task Changes                                 | CI Changes                                                              | Notes                             |
| :---------------- | :---------------------------------- | :------------------------------------------- | :---------------------------------------------------------------------- | :-------------------------------- |
| ai-mktpl          | Already `.mise.toml`                | Add fix, lint, check to `.mise/tasks/`       | Create `check.yaml` with auto-fix; retire existing CI if redundant      | Leave justfile as-is              |
| .github           | Already `.mise.toml`                | Move/rename tasks to `.mise/tasks/`          | Rename `ci.yaml` -> `check.yaml`, add auto-fix                         | Has many custom tasks; keep them  |
| aitkit            | Already `.mise.toml`                | Add fix, lint, test, check to `.mise/tasks/` | Already has auto-format; rename `lint.yml`+`test.yml` -> `check.yaml`   | Go project                        |
| agent-team        | `mise.toml` -> `.mise.toml`         | Rename format->fix; move to `.mise/tasks/`   | Rename `test.yaml` -> `check.yaml`, add auto-fix                       | PR #97 needs update               |
| claude-utils      | `mise.toml` -> `.mise.toml`         | Add fix, check; move to `.mise/tasks/`       | Rename `test.yaml` -> `check.yaml`, add auto-fix                       |                                   |
| op-exec           | `mise.toml` -> `.mise.toml`         | Add fix, test, check; move to `.mise/tasks/` | Rename `test.yaml` -> `check.yaml`, add auto-fix                       |                                   |
| git-wt            | `mise.toml` -> `.mise.toml`         | Add fix, check; move to `.mise/tasks/`       | Rename `test.yaml` -> `check.yaml`, add auto-fix                       |                                   |
| homebrew-devsetup | `mise.toml` -> `.mise.toml`         | Move to `.mise/tasks/`; rename format->fix   | Already has `check.yaml` with auto-format                               | Gold standard for CI pattern      |
| github-actions    | `mise.toml` -> `.mise.toml`         | Move to `.mise/tasks/`; verify names         | Already has `check.yaml`                                                | May need minor updates            |
| renovate-config   | Already `.mise.toml`                | Add fix, lint, check to `.mise/tasks/`       | Rename `check.yml` -> `check.yaml`                                      | Has `sync-json.yml` too           |
| .org              | N/A                                 | N/A                                          | N/A                                                                     | Minimal repo, exempt              |

## Existing Draft PRs

These PRs were created before this spec was finalized and need revision to match the updated standard:

**Phase 1 — Filename renames (direction reversed — now renaming TO `.mise.toml`):**

- [ai-mktpl#189](https://github.com/nsheaps/ai-mktpl/pull/189) — **close**: was renaming wrong direction; already uses `.mise.toml`
- [.github#18](https://github.com/nsheaps/.github/pull/18) — **close**: already uses `.mise.toml`
- [aitkit#12](https://github.com/nsheaps/aitkit/pull/12) — **close**: already uses `.mise.toml`

**Phase 2 — Task standardization (needs update: `format` -> `fix`, inline -> `.mise/tasks/`):**

- [agent-team#97](https://github.com/nsheaps/agent-team/pull/97) — **needs update**: uses `format` not `fix`, inline not file-based, modifies `test.yaml` not `check.yaml`
- [claude-utils#6](https://github.com/nsheaps/claude-utils/pull/6) — **needs update**: uses `format` not `fix`, inline not file-based
- [op-exec#1](https://github.com/nsheaps/op-exec/pull/1) — **needs update**: uses `format` not `fix`, inline not file-based
- [git-wt#10](https://github.com/nsheaps/git-wt/pull/10) — **needs update**: uses `format` not `fix`, inline not file-based
- [homebrew-devsetup#103](https://github.com/nsheaps/homebrew-devsetup/pull/103) — **needs update**: uses `format` not `fix`, inline not file-based

**Phase 3 — CI workflow standardization (not yet started):**

To be created after spec approval:

- Rename `test.yaml` -> `check.yaml` in agent-team, claude-utils, op-exec, git-wt
- Rename `check.yml` -> `check.yaml` in renovate-config
- Add auto-fix+commit+fail pattern to all `check.yaml` workflows
- Add `.github/actions/github-app-auth/action.yml` composite action to repos that don't have it
- Map secrets to env vars (`GH_APP_ID`, `GH_APP_PRIVATE_KEY`) at workflow level
- Add `renovate.json5` to repos that don't have it

## References

- [mise Documentation](https://mise.jdx.dev/)
- [mise Configuration](https://mise.jdx.dev/configuration.html)
- [mise File Tasks](https://mise.jdx.dev/tasks/file-tasks.html)
- [stefanzweifel/git-auto-commit-action](https://github.com/stefanzweifel/git-auto-commit-action)
- [actions/create-github-app-token](https://github.com/actions/create-github-app-token)
- [jdx/mise-action](https://github.com/jdx/mise-action)
- [nsheaps/renovate-config](https://github.com/nsheaps/renovate-config)
- [Conventional Commits](https://www.conventionalcommits.org/)
- [release-it Documentation](https://github.com/release-it/release-it)
- Audit report: `nsheaps/agent-team:.claude/tmp/mise-consistency-audit.md`
- Draft proposal: `nsheaps/agent-team:.claude/tmp/mise-standard-proposal.md`
