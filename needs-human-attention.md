# Needs Human Attention

Consolidated list of items across the nsheaps org that require the repo owner's manual intervention — credentials, decisions, approvals, or configuration that cannot be automated.

Generated: 2026-02-23 (looney-toons-20260223 session)

---

## P1 — Credential Setup

These items are blocked until the owner creates credentials and configures secrets.

| Item | Repo | What's Needed | Tracking Issue |
| :--- | :--- | :--- | :--- |
| GitHub App secrets | agent-team | Create AUTOMATION_GITHUB_APP_ID and AUTOMATION_GITHUB_APP_PRIVATE_KEY, add as repo secrets | [agent-team#4](https://github.com/nsheaps/agent-team/issues/4) |
| GitHub App secrets | agent | Same credentials as above, add to this repo | [agent#9](https://github.com/nsheaps/agent/issues/9) |
| 1Password Service Account | .github | Create a CI/CD service account in 1Password admin console for the secret sync workflow | [.github#8](https://github.com/nsheaps/.github/issues/8) |
| GitHub PAT for secret sync | .github | Create a PAT with `repo` scope for `gh secret set` in the sync workflow | [.github#8](https://github.com/nsheaps/.github/issues/8) |
| Ansible secret sync | .github | Authorize and configure Ansible playbook for cross-repo secret sync | [.github#11](https://github.com/nsheaps/.github/issues/11) |
| op-exec release secrets | op-exec | GitHub App secrets needed for release pipeline (Foghorn's extraction in progress) | Task #8 (session) |

## P1 — Owner Decisions

These items require a policy or design decision before work can proceed.

| Item | Repo | Decision Needed | Tracking Issue |
| :--- | :--- | :--- | :--- |
| License mismatch | agent-team | MIT file vs UNLICENSED in package.json vs Proprietary in README — which is correct? | [agent-team#8](https://github.com/nsheaps/agent-team/issues/8) |
| License mismatch | agent | UNLICENSED in package.json vs MIT in LICENSE file — which is correct? | [agent#7](https://github.com/nsheaps/agent/issues/7) |
| 60MB dist/ binary in git | agent | Rewrite git history to remove it, or add to .gitignore and accept the bloat? | [agent#4](https://github.com/nsheaps/agent/issues/4) |
| sync-settings.py race condition | ai-mktpl | File locking bug with potential data loss — needs reproduction and decision on fix approach | [ai-mktpl#166](https://github.com/nsheaps/ai-mktpl/issues/166) |

## P2 — Infrastructure Decisions

These require owner input on org-wide policy or architecture.

| Item | Repo | Decision Needed | Tracking Issue |
| :--- | :--- | :--- | :--- |
| Homebrew publishing | agent-team | Should agent-team publish to Homebrew tap? | [agent-team#5](https://github.com/nsheaps/agent-team/issues/5) |
| Rule authority | agent-team | Where should verify-before-blaming rule live? (agent-team vs ai-mktpl vs both) | [agent-team#9](https://github.com/nsheaps/agent-team/issues/9) |
| Settings sync | .github | Design org-wide settings sync infrastructure | [.github#9](https://github.com/nsheaps/.github/issues/9) |
| GitHub App sync | .github | Design org-wide GitHub App sync infrastructure | [.github#10](https://github.com/nsheaps/.github/issues/10) |
| Priority label sync | .github | Automate syncing p1-p4 labels across all org repos — org-wide policy decision | [.github#12](https://github.com/nsheaps/.github/issues/12) |
| 1Password vault structure | (org-wide) | Single CI/CD vault vs per-environment vaults — security architecture decision | See [1Password research](https://github.com/nsheaps/agent-team/blob/main/docs/research/1password-secrets-sync.md) |

## Session-Specific Items

Items from the 2026-02-23 session that need human attention.

| Item | Status | What's Needed |
| :--- | :--- | :--- |
| Settings.json recovery | Resolved | Backup restored from `/Users/nathan.heaps/.claude/backups/2026-02-23/settings.json` |
| Statusline plugin fix | PR open | [ai-mktpl#176](https://github.com/nsheaps/ai-mktpl/pull/176) — fixes root cause of settings.json truncation. Needs merge. |
| github-actions repo split | Pending | Create issue to split nsheaps/github-actions into individual action repos | Task #72 (session) |

## Consolidation Opportunities

Items that can be batched for efficiency:

1. **License sweep**: [agent-team#8](https://github.com/nsheaps/agent-team/issues/8) + [agent#7](https://github.com/nsheaps/agent/issues/7) — decide license once, apply to all repos
2. **GitHub App secrets**: [agent-team#4](https://github.com/nsheaps/agent-team/issues/4) + [agent#9](https://github.com/nsheaps/agent/issues/9) + [.github#11](https://github.com/nsheaps/.github/issues/11) — create credentials once, sync to all repos via the sync workflow
3. **GITHUB_JOB_URL bug**: [claude-team#1](https://github.com/nsheaps/claude-team/issues/1) + [gs-stack-status#4](https://github.com/nsheaps/gs-stack-status/issues/4) — same root cause, fix once

## References

- [Issue Triage Report](https://github.com/nsheaps/agent-team/blob/main/.claude/scratch/issue-triage.md) — full 89-issue audit
- [1Password Secrets Sync Research](https://github.com/nsheaps/agent-team/blob/main/docs/research/1password-secrets-sync.md) — best practices and recommendations
- [AI Agent Eng Failure Log](https://github.com/nsheaps/agent-team/blob/main/.claude/tmp/ai-agent-eng-failure-log.md) — session failure patterns
