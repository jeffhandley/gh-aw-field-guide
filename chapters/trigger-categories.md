---
title: "The Two Categories of Triggers"
---

[← Previous: workflow_run](../triggers/workflow-run.md) | [Table of Contents](../README.md) | [Next: Apparent vs Actual →](apparent-vs-actual.md)

# The Two Categories of Triggers

## Standard GitHub Actions events

Every gh-aw workflow ultimately compiles to a standard `on:` block. The full set, per GitHub's reference[^events-docs]:

| Event | Activity types (where applicable) | Runs on default branch only? | Notable security/behavioral notes |
|---|---|---|---|
| `branch_protection_rule` | created, edited, deleted | Yes | Admin-only causes; safe |
| `check_run` | created, rerequested, completed, requested_action | Yes | Doesn't recurse on Actions-created check suites |
| `check_suite` | completed | Yes | Same |
| `create` | (none) | No | Branch/tag created; no payload secrets risk |
| `delete` | (none) | No | Branch/tag deleted |
| `deployment` | (none) | No | Triggered by API/UI deployment |
| `deployment_status` | (none) | No | |
| `discussion` | created, edited, deleted, transferred, pinned, unpinned, labeled, unlabeled, locked, unlocked, category_changed, answered, unanswered | Yes | Repo must have Discussions enabled |
| `discussion_comment` | created, edited, deleted | Yes | Read-only users **can** fire |
| `fork` | (none) | Yes | Fires when *anyone* (incl. outside) forks |
| `gollum` | (none) | Yes | Wiki edits |
| `issue_comment` | created, edited, deleted | Yes | Fires for **both** issue *and* PR comments; PR distinction via `github.event.issue.pull_request != null` |
| `issues` | opened, edited, deleted, transferred, pinned, unpinned, closed, reopened, assigned, unassigned, labeled, unlabeled, locked, unlocked, milestoned, demilestoned, typed, untyped | Yes | Read-only users can `opened`, `edited` (own), `closed` (own), `reopened` (own) |
| `label` | created, edited, deleted | Yes | Repo-level label CRUD, not application |
| `member` | added, edited, removed | Yes | |
| `merge_group` | checks_requested | Yes | Merge queue checks |
| `milestone` | created, closed, opened, edited, deleted | Yes | |
| `page_build` | (none) | Yes | GitHub Pages build |
| `public` | (none) | Yes | Repo went public |
| `pull_request` | opened, edited, closed, assigned, unassigned, review_requested, review_request_removed, ready_for_review, labeled, unlabeled, synchronize, converted_to_draft, locked, unlocked, enqueued, dequeued, milestoned, demilestoned, reopened, auto_merge_enabled, auto_merge_disabled | Workflow runs on PR's merge ref or head ref | **Read-only `GITHUB_TOKEN`, no secrets** when PR is from a fork[^token-permissions] |
| `pull_request_review` | submitted, edited, dismissed | (PR ref) | |
| `pull_request_review_comment` | created, edited, deleted | (PR ref) | |
| `pull_request_target` | (same as `pull_request`) | **Default branch, with secrets and write token** | **Most dangerous trigger** — never check out PR head SHA without sandboxing[^pwn-requests] |
| `push` | (none, but supports `branches`/`paths`/`tags` filters) | (the pushed ref) | Forks cannot trigger (they push to their fork) |
| `registry_package` | published, updated | Yes | |
| `release` | published, unpublished, created, edited, deleted, prereleased, released | Yes | |
| `repository_dispatch` | (custom `event_type`) | Yes | API-triggered with PAT having `repo` scope |
| `schedule` | (cron) | Yes | Disabled after 60 days of repo inactivity |
| `status` | (none) | Yes | Commit status changes |
| `watch` | started | Yes | Repo starred |
| `workflow_call` | (called from another workflow) | (caller's ref) | Reusable workflow |
| `workflow_dispatch` | (manual) | (selected ref) | **Requires write access** to the repo to invoke[^approval-docs] |
| `workflow_run` | requested, completed, in_progress | Default branch | Runs with **base repo's** secrets even when triggered by a fork PR's CI run — second-most-dangerous trigger[^pwn-requests] |

> Several "deprecated"/legacy events still exist (`project`, `project_card`, `project_column` for classic projects). They are not currently supported by gh-aw shorthands but accept-as-passthrough in `on:`.

## gh-aw "convenience" / "virtual" triggers

gh-aw calls these **"trigger types"** but the docs themselves repeatedly note they expand to standard events plus filtering[^triggers-doc][^command-doc]. The framework's preferred terminology is **"convenience trigger"** for `slash_command:` / `label_command:` and **"trigger shorthand"** for the natural-language strings (`on: push to main`, `on: issue labeled bug`, etc.).

| gh-aw trigger | Compiles to | Filtered by what (in activation job) |
|---|---|---|
| `slash_command: name` | `issues:[opened,edited,reopened]` + `issue_comment:[created,edited]` + `pull_request:[opened,edited,reopened]` + `pull_request_review_comment:[created,edited]` + `discussion:[created,edited]` + `discussion_comment:[created,edited]` + `workflow_dispatch` (default; narrow with `events:`) | First-word match of `/<name>` in body/comment text |
| `label_command: name` | `issues:[labeled]` + `pull_request:[labeled]` + `discussion:[labeled]` + `workflow_dispatch` (default; narrow with `events:`) | Label name match; auto-removes label after activation (unless `remove_label: false`) |
| `reaction: <emoji>` | (additive) Posts an emoji on the triggering item from the activation job | n/a |
| `status-comment: true` | (additive) Posts a "started/completed" comment on the triggering item | n/a |
| `forks: ["pattern"]` | (additive) Allow PRs from matching forks; default is **block all forks** | Repository-ID comparison in activation job |
| `names: [labels]` | (additive) Filter `issues`/`pull_request` events by label name | Label list contains in activation job |
| `lock-for-agent: true` | (additive) Locks the parent issue at start, unlocks at end | n/a |
| `on.roles: [admin, maintainer, write] \| all` | (additive) **Allowlist** of repo roles allowed to trigger. **Default `[admin, maintainer, write]` is injected automatically** when the workflow has any "unsafe" event (issues, comments, PRs, discussions, slash/label commands). `roles: all` disables the check[^role-checks-go]. | Membership API call in activation job |
| `skip-roles: [role,...]` | (additive) **Blocklist** — skip when actor has any of the listed repo roles. Layered on top of `on.roles:` | Membership API call in activation job |
| `skip-bots: [actor,...]` | (additive) Skip when actor matches | Actor name match in activation job |
| `skip-if-match: query` | (additive) Skip when GitHub Search query has matches (≥ `max`) | Search API call in activation job |
| `skip-if-no-match: query` | (additive) Skip when query has fewer than `min` matches | Same |
| `manual-approval: env-name` | (additive) Sets `environment:` on activation job → environment protection rules | GitHub-native env approval gate |
| `stop-after: +25h` or date | (additive) Activation job exits after deadline; recompile resets | n/a |
| `on.steps: [...]` | (additive) Custom deterministic steps in pre-activation job | Auto-wired `<id>_result` outputs |
| `on.permissions: {...}` | (additive) Extra `GITHUB_TOKEN` scopes for pre-activation job | n/a |
| `on.github-token: ...` / `on.github-app: ...` | (additive) Custom token/app for activation job (reactions, status comments, skip-if searches) | n/a |
| Schedule shorthands (`daily`, `weekly on monday`, `every 10 minutes`, `daily around 14:00`, `daily between 9:00 and 17:00`) | `schedule: [cron]` (compiler scatters minute offsets to spread load) | n/a |
| Trigger shorthands (`on: push to main`, `on: pull_request opened`, `on: issue labeled bug`, `on: repository starred`, `on: dependabot pull request`, `on: api dispatch custom-event`, `on: manual`, `on: comment created`, `on: workflow completed ci-test`, `on: security alert`, etc.) | Standard event(s) + auto-injected `workflow_dispatch` | Whatever the shorthand expands to |
| `on: /my-bot` (ultra-short) | `slash_command: my-bot` + `workflow_dispatch` | First-word match |

**Critical insight.** Every gh-aw "virtual" trigger results in **the workflow actually starting** when *any* of the underlying standard events fires. The "filtering" all happens in a real GitHub Actions job — the **pre-activation / activation job** — which then either (a) sets a job output that the agent job's `if:` consumes, or (b) early-exits and lets the agent job be skipped[^triggers-doc]. This means:

- A "skipped" run still appears in the Actions tab.
- It still consumes a runner minute (small, but nonzero — the activation job runs on `ubuntu-latest`).
- It still participates in `concurrency.group` and can **cancel** in-progress siblings.
- It still counts against any per-repo workflow concurrency / queue limits.

📚 See [Appendix B: Footnotes](../appendices/footnotes.md) for source citations.

[^triggers-doc]: gh-aw [Triggers reference](https://github.github.com/gh-aw/reference/triggers/)
[^command-doc]: gh-aw [Command Triggers reference](https://github.github.com/gh-aw/reference/command-triggers/)
[^role-checks-go]: gh-aw source, [`pkg/workflow/role_checks.go`](https://github.com/github/gh-aw/blob/main/pkg/workflow/role_checks.go)
[^approval-docs]: GitHub Docs, [Approving workflow runs from public forks](https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-workflow-runs/approving-workflow-runs-from-public-forks)
[^token-permissions]: GitHub Docs, [Automatic token authentication](https://docs.github.com/en/actions/security-guides/automatic-token-authentication)
[^pwn-requests]: GitHub Security Lab, [Keeping your GitHub Actions and workflows secure: Preventing pwn requests](https://securitylab.github.com/research/github-actions-preventing-pwn-requests/)

---

[← Previous: workflow_run](../triggers/workflow-run.md)| [Table of Contents](../README.md) | [Next: Apparent vs Actual →](apparent-vs-actual.md)
