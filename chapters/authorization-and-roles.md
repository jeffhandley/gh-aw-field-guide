<nav>

[← Previous: Concurrency and Race Conditions](concurrency-and-races.md) | [Table of Contents](../README.md) | [Next: Appendix A — Trigger-by-Trigger Risk Profile →](../appendices/trigger-risk-profile.md)

</nav>

# Authorization, Roles, and Read-Only Contributors

The emphasis on **read-only contributors** is critical, and the answer has two layers: **what GitHub itself permits** and **what gh-aw layers on top by default**.

## What each role can do *that fires a workflow* (raw GitHub permissions)

Here is what each role can do *that fires a workflow* — at the GitHub event layer, before any gh-aw filtering:

| Action | Read | Triage | Write | Maintain | Admin |
|---|---|---|---|---|---|
| Open an issue | ✅ | ✅ | ✅ | ✅ | ✅ |
| Edit own issue body/title | ✅ | ✅ | ✅ | ✅ | ✅ |
| Close own issue | ✅ | ✅ | ✅ | ✅ | ✅ |
| Comment on issue/PR/discussion | ✅ | ✅ | ✅ | ✅ | ✅ |
| Edit own comment | ✅ | ✅ | ✅ | ✅ | ✅ |
| React with emoji | ✅ | ✅ | ✅ | ✅ | ✅ |
| Open a PR (from fork or, if collaborator, from branch) | ✅ (via fork) | ✅ | ✅ | ✅ | ✅ |
| Edit own PR body | ✅ | ✅ | ✅ | ✅ | ✅ |
| `/slash-command` in any comment/body they author | ✅ | ✅ | ✅ | ✅ | ✅ |
| Apply a label | ❌ | ✅ | ✅ | ✅ | ✅ |
| Approve/dismiss reviews | ❌ | ❌ | ✅ | ✅ | ✅ |
| Push to repo branches | ❌ | ❌ | ✅ | ✅ | ✅ |
| Invoke `workflow_dispatch` | ❌ | ❌ | ✅ | ✅ | ✅ |
| Click "Approve and run workflows" | ❌ | ❌ | ✅ | ✅ | ✅ |

**Key observation:** A reader can fire any [`slash_command:`](../triggers/comment-and-slash-command.md) and any `issues`/`issue_comment`/`pull_request`/`discussion_comment`/`reaction`-driven workflow, just by typing or clicking. They cannot fire `label_command:` or `workflow_dispatch:` triggered ones.

## gh-aw's `on.roles:` allowlist (the *primary* authz mechanism)

gh-aw automatically injects an **activation-job membership check** into any workflow whose triggers include "unsafe" events (issues, comments, PRs, discussions, slash/label commands). The allowlist is configured by `on.roles:` and **defaults to `[admin, maintainer, write]`**[^role-checks-go]. Behaviors:

- **String form** — `on.roles: write` (single role) or `on.roles: all` (special value disabling the check entirely).
- **Array form** — `on.roles: [admin, maintainer, write]` (the default) or `on.roles: [admin, maintainer, write, triage, read]` to broaden.
- **Inverse field** — `on.skip-roles: [role,...]` blocks specific roles from a wider allowlist (e.g., `on.roles: all` + `on.skip-roles: [read]`).
- **The check still consumes a runner.** Like other activation-job filters, the workflow *runs* and the activation job calls the GitHub repo permission API; the agent job is then skipped if the actor's role is not in the allowlist.
- **`on.roles: all` is required for any chat-style workflow that intentionally serves read-only contributors** — for example, a self-service `/help` command or a community FAQ bot.
- **Does not protect against [`workflow_run`](../triggers/workflow-run.md) chained workflows** (which run as the base repo, not as the original actor).

## GitHub permission roles (vocabulary for this section)

| Role | Default in `on.roles`? | Notes |
|---|---|---|
| `admin` | ✅ | Full control |
| `maintainer` (a.k.a. `maintain`) | ✅ | Can manage repo without billing/admin |
| `write` | ✅ | Can push, label, approve PRs, dispatch workflows |
| `triage` | ❌ | Can manage issues/PRs (label, assign, close) but cannot push |
| `read` | ❌ | Can open issues/PRs/comments/reactions only |
| (anonymous fork contributor) | ❌ | Treated as `read` for membership API purposes |
| `bot` actor | (inherits the bot's permission level) | Use `on.skip-bots:` to block specific bot logins |

**`triage` is invisible in the default allowlist** — that means a workflow like [`label_command:`](../triggers/labeled-and-label-command.md) (which requires triage to apply the label in the first place) will *fire* but its activation job will *deny* a triage user unless `on.roles:` is broadened. This is a frequent footgun.

## Consolidated read-only / fork contributor write surface 🛑

The single most important security insight in this guide:

> **What the agent can do is determined by the workflow's `permissions:` and `safe-outputs:` declarations — NOT by the actor who fired it.** When a workflow accepts a read-only or fork contributor as the trigger (i.e., `on.roles: all`, no actor check, or no fork guard), that contributor effectively gets **bot-level write access** to anything the workflow grants the agent.

This is why the gh-aw `on.roles:` default of `[admin, maintainer, write]` exists — it's a *deny-by-default* gate that prevents a read user from inducing the bot to act with the bot's elevated permissions. The moment you set `on.roles: all` (or write a workflow that ignores actor identity), every mutation in the table below becomes reachable by anyone who can fire the trigger.

**Capability matrix.** Assuming the workflow grants the relevant `permissions:` and exposes the relevant safe-output, can a **read-only contributor** (or anonymous fork contributor) cause the listed action by firing each gh-aw trigger?

Legend: ✅ = reachable; ⚠️ = reachable but with caveats; ❌ = not reachable via this trigger; 🛑 = reachable AND the workflow runs with **upstream secrets** (the high-impact pwn vector).

| Action the agent can take | `slash_command` on issue/comment | `slash_command` on own PR (cross-fork) | `issues` (opened/edited) | `issue_comment` (PR comment, cross-fork) | `discussion` / `discussion_comment` | `pull_request` (cross-fork) | `pull_request_target` (cross-fork) |
|---|---|---|---|---|---|---|---|
| Post a comment on the triggering item | ✅ | ✅ | ✅ | 🛑 | ✅ | ⚠️ read-only token | 🛑 |
| Add or remove labels (`add-labels`) | ✅ | ✅ | ✅ | 🛑 | ✅ | ⚠️ | 🛑 |
| Edit issue/PR title or body (`update-issue`) | ✅ | ✅ | ✅ | 🛑 | ❌ (discussions don't have an analog safe-output) | ⚠️ | 🛑 |
| Close / reopen issue or PR (`update-issue`) | ✅ | ✅ | ✅ | 🛑 | ❌ | ⚠️ | 🛑 |
| Edit or delete *existing* comments | ❌ no safe-output for editing/deleting comments | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Create a new issue (`create-issue`) | ✅ | ✅ | ✅ | 🛑 | ✅ | ⚠️ | 🛑 |
| Create a new pull request (`create-pull-request`) | ✅ | ✅ | ✅ | 🛑 | ✅ | ⚠️ | 🛑 |
| Push commits to the PR branch (`push-to-pull-request-branch`) | ❌ (no PR context) | ⚠️ only when PR is from same repo, not cross-fork | ❌ | ⚠️ same-repo only | ❌ | ❌ cross-fork is blocked | ⚠️ same-repo only |
| Edit code on the PR (commit changes) | ❌ | ⚠️ same caveat as push | ❌ | ⚠️ same | ❌ | ❌ | ⚠️ same |
| Push to the upstream default branch | ❌ unless workflow runs `git push` from a `bash` step (very ill-advised) | same | same | same | same | same | same |
| Create a discussion (`create-discussion`) | ✅ | ✅ | ✅ | 🛑 | ✅ | ⚠️ | 🛑 |
| Trigger a downstream `workflow_dispatch` (via `gh workflow run`) | ✅ if `permissions: actions: write` | same | same | 🛑 | same | ⚠️ | 🛑 |
| Read repository secrets (and exfiltrate via comment) | ✅ — secrets are in the agent's `env`, can be echoed into any output | same | same | 🛑 | same | ❌ no secrets on fork PRs | 🛑 |
| Approve a pull request | ❌ — a workflow's `GITHUB_TOKEN` cannot approve PRs (GitHub policy) | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Merge a pull request | ✅ if `permissions: pull-requests: write, contents: write` and PR is mergeable | same | same | 🛑 | n/a | ⚠️ | 🛑 |
| Add/remove repo collaborators, change branch protection | ❌ — `GITHUB_TOKEN` lacks admin scopes; requires a separate PAT | same | same | same | same | same | same |

**How to read the 🛑 column intersections.** A 🛑 means: a contributor with **only read access** (or none — anonymous fork contributors via PR comments) can, just by typing a `/command` or comment, induce the bot to perform the listed mutation **using the upstream's secrets and write token**. This is the gh-aw-flavored re-statement of the classic "pwn requests" class[^pwn-requests] and is the entire reason `on.roles:` defaults to `[admin, maintainer, write]`.

**Defenses, in priority order:**

1. **Leave `on.roles:` at its default.** Don't set `on.roles: all` unless you've thought through every cell of the matrix above for your workflow's actual `permissions:` and safe-outputs.
2. **Minimize `permissions:`** to the smallest set the agent actually needs.
3. **Minimize `safe-outputs:`** to only the structured mutations the workflow needs to make.
4. **For PR-touching workflows: never check out the PR head SHA in the same job that has secrets.** Use `pull_request` (read-only token) for code analysis, and a separate `pull_request_target` job *without* checkout for the comment/label posting.
5. **Add an explicit fork guard** (`if: github.event.pull_request.head.repo.fork == false`) for any agent step that should refuse to act on cross-fork PRs.

📚 See [Appendix B: Footnotes](../appendices/footnotes.md) for source citations.

---

<nav>

[← Previous: Concurrency and Race Conditions](concurrency-and-races.md) | [Table of Contents](../README.md) | [Next: Appendix A — Trigger-by-Trigger Risk Profile →](../appendices/trigger-risk-profile.md)

</nav>
