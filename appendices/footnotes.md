---
title: "Footnotes"
---

[← Previous: Appendix A — Trigger-by-Trigger Risk Profile](trigger-risk-profile.md) | [Table of Contents](../README.md) | Next →

# Appendix B: Footnotes

Sources cited throughout the Field Guide. Each entry's anchor matches the `[^name]` reference used in the chapters.

---

<a id="triggers-doc"></a>
**[^triggers-doc]** &nbsp; gh-aw **Triggers** reference — `https://raw.githubusercontent.com/githubnext/gh-aw/main/docs/src/content/docs/reference/triggers.md` (fetched 2026-04-17). Sections cited: *Trigger Types*, *Command Triggers (`slash_command:`)*, *Label Command Trigger (`label_command:`)*, *Reactions*, *Status Comments*, *Stop After Configuration*, *Manual Approval Gates*, *Skip-If-Match*, *Skip-If-No-Match*, *Pre-Activation Steps*, *Trigger Shorthands*.

<a id="command-doc"></a>
**[^command-doc]** &nbsp; gh-aw **Command Triggers** reference — `https://raw.githubusercontent.com/githubnext/gh-aw/main/docs/src/content/docs/reference/command-triggers.md` (fetched 2026-04-17). Quoted: *"By default, command triggers listen to all comment-related events, which can create skipped runs in the Actions UI"*; *"The command must be the first word of the comment or body text to trigger the workflow"*; supported events list (`issues`, `issue_comment`, `pull_request`, `pull_request_comment`, `pull_request_review_comment`, `discussion`, `discussion_comment`, `*`).

<a id="events-docs"></a>
**[^events-docs]** &nbsp; GitHub Docs, **Events that trigger workflows** — `https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows` (fetched 2026-04-17). Source for the standard event table including activity types, "default branch only" flag, and the schedule-disabled-after-60-days rule.

<a id="frontmatter-doc"></a>
**[^frontmatter-doc]** &nbsp; gh-aw **Frontmatter** reference — `https://raw.githubusercontent.com/github/gh-aw/main/docs/src/content/docs/reference/frontmatter.md` (fetched 2026-04-17). Lists `skip-roles`, `skip-bots`, `skip-if-match`, `skip-if-no-match`, `manual-approval`, `forks`, `stop-after`, `reaction`, `status-comment`, `on.steps`, `on.permissions`, `on.github-token`, `on.github-app`. (Note: the rendered frontmatter doc does not currently list `on.roles:`; that field is documented authoritatively in the source code — see the next entry.)

<a id="role-checks-go"></a>
**[^role-checks-go]** &nbsp; gh-aw source — [github/gh-aw/`pkg/workflow/role_checks.go`](https://github.com/github/gh-aw/blob/main/pkg/workflow/role_checks.go). Authoritative for the `on.roles:` semantics: default allowlist `["admin", "maintainer", "write"]` (line ~109), special string value `"all"` disables the check (line ~118), per-event applicability via `needsRoleCheck` (line ~300), `hasSafeEventsOnly` short-circuit for safe-only triggers (line ~325). Also documents the parallel `ignored-roles:` field (default `["admin", "maintain", "write"]`) used for activation rate-limit exemption.

<a id="approval-docs"></a>
**[^approval-docs]** &nbsp; GitHub Docs, **Approving workflow runs from public forks** — `https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-workflow-runs/approving-workflow-runs-from-public-forks`. Describes the *Approve and run workflows* button and the conditions under which it appears.

<a id="approval-policy"></a>
**[^approval-policy]** &nbsp; GitHub Docs, **Disabling or limiting GitHub Actions for your repository / organization → Approving workflow runs from public forks** — `https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository`. Source of the four approval-policy levels (first-time-contributor-new-to-GitHub, first-time-contributor, all outside collaborators, all external contributors).

<a id="token-permissions"></a>
**[^token-permissions]** &nbsp; GitHub Docs, **Automatic token authentication → Permissions for the GITHUB_TOKEN** — `https://docs.github.com/en/actions/security-guides/automatic-token-authentication`. Source for the read-only-and-no-secrets behavior on fork PRs under `pull_request`.

<a id="pwn-requests"></a>
**[^pwn-requests]** &nbsp; GitHub Security Lab, **Keeping your GitHub Actions and workflows secure: Preventing pwn requests** — `https://securitylab.github.com/research/github-actions-preventing-pwn-requests/`. Authoritative analysis of `pull_request_target`, `workflow_run`, and `issue_comment`-from-fork attack patterns.

<a id="concurrency"></a>
**[^concurrency]** &nbsp; GitHub Docs, **Using concurrency, expressions, and a test matrix → Using concurrency** — `https://docs.github.com/en/actions/using-jobs/using-concurrency`. Describes group-keyed serialization and `cancel-in-progress` semantics; cancellation is asynchronous (SIGTERM → 7500 ms grace → SIGKILL).

---

[← Previous: Appendix A — Trigger-by-Trigger Risk Profile](trigger-risk-profile.md) | [Table of Contents](../README.md) | Next →
