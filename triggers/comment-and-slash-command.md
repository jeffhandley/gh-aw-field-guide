<nav>

[← Previous: `push`](push.md) | [Table of Contents](../README.md) | [Next: `discussion` →](discussion.md)

</nav>

# `issue_comment` and `slash_command:`

### ⚠️ Use with caution

> 🛑 **A. The "no approval gate" loophole.** `issue_comment` does **NOT** trigger the "Approve and run workflows" gate, even when fired from a comment on a fork PR. A fork contributor can fire any `issue_comment`-listening workflow on their own PR with no maintainer interaction. This is the canonical reason `slash_command:` workflows have a much wider invocation surface than they appear — `slash_command:` subscribes to `issue_comment` by default.

> 🛑 **B. Issues vs. PRs naming trap.** Comments on PRs fire `issue_comment`, **NOT** `pull_request_review_comment` (which is a different event for inline review threads). Workflows authored as "an issue triager" inadvertently respond to comments on PRs from forks — even though the author never put `pull_request*` in `on:`. Distinguish via `github.event.issue.pull_request != null`.

> 🛑 **C. Edits to old comments fire NOW.** An attacker can edit a 6-month-old comment on a closed issue or PR, injecting `/command` or any payload — `issue_comment.edited` fires today against today's secrets, today's `permissions:`, today's `safe-outputs:` allowlist. The workflow has no concept of "this comment was created when our security model was different." [Tenet #10](../chapters/tenets.md)'s worst-case invocation rate must include the attacker reaching back through the entire comment history of the repo.

> 🛑 **`slash_command:` is the trigger most users mentally model as "fires only when someone with the right role types `/my-command`" — which is wrong by an order of magnitude or more, and the framing leads directly to the concurrency catastrophe.** What `slash_command:` actually does is subscribe to a *broad set of underlying events* (every issue comment, every PR review comment, every issue/PR open/edit, every discussion comment) and then filter at activation time. **Every one of those events spawns a workflow run** that consumes a runner slot and a brief execution context — only the `/command` filter and the `roles:` filter cause an early abort. Most workflow authors never see this because the runs abort fast and the UI lists "Skipped" — but they **did run**, they **did consume the runner**, and they **did affect concurrency groups**.
>
> *Idempotency requirement:* because concurrency groups cannot be used safely with `slash_command:`, slash-command workflows **MUST be implemented to be idempotent** — by implementing the same concurrency checks or locking in code that schedule-jobs-that-poll must employ. Treat every `/command` activation as if the same `/command` might already be running for the same target: check before acting, claim a lock (e.g., via a label, an issue-comment marker, a check-run with a unique name, or external state), no-op if the work is already in progress or already done, and release/clean-up on completion. The discipline is the same as for scheduled pollers, and the failure modes when you skip it are the same: duplicate comments, duplicate labels, duplicate downstream effects, and races that leave the target in an indeterminate state.

---

## Scenarios

- On-demand agentic operations via `/command` comments — `/triage`, `/review`, `/deploy`, `/help`
- Responding to questions or requests in issue/PR conversations
- **The preferred trigger for PR operations that need write access** — not subject to the "Approve and run workflows" button, even on fork PRs
- Community-facing self-service bots (with `on.roles: all` and appropriate safe-outputs)
- Distinguish issue comments from PR comments via `github.event.issue.pull_request != null`

**Why ⚠️:** `slash_command:` subscribes to a broad set of underlying events by default (every issue comment, PR review comment, issue/PR open/edit, discussion comment). Every one of those events spawns a workflow run that consumes a runner and participates in concurrency groups. The concurrency catastrophe (non-matching comment cancels in-flight `/command`) is the headline risk. Use `events:` to narrow the subscription.

## Profile

| Dimension | Recommendation |
|---|---|
| `on.roles:` | Default `[admin, maintainer, write]` for privileged operations. `triage` excluded by default — add when triage users are the primary operators. `all` for community-facing commands — audit every safe-output cell in the [capability matrix](../chapters/authorization-and-roles.md) first. |
| Activity types | For raw `issue_comment`: `[created]` is safest. Add `edited` only if you've designed for the time-bomb (edits to old comments on closed issues/PRs fire today with today's secrets). For `slash_command:`: **always narrow `events:`** — e.g., `events: [issue_comment]` when the command only appears in comments. Default subscribes to everything. |
| Concurrency | `${{ github.workflow }}-${{ github.event.issue.number }}`. Use `cancel-in-progress: false` to prevent a benign drive-by, automated, or even filtered comment event from canceling a legitimate in-flight workflow run. |
| Idempotency | **Required.** Treat every activation as if the same command might already be running. Check before acting, claim a lock, no-op if already in progress or done. gh-aw provides `lock-for-agent: true` to automatically lock and unlock the issue during execution, but use with extreme caution — it prevents genuine users from interacting on the issue/PR while the workflow is running. |
| Fork posture | Apply `if: ${{ github.event_name == 'workflow_dispatch' \|\| !github.event.repository.fork }}` to prevent running within a user's fork. |
| Approval gate | **Not subject** to the "Approve and run workflows" button — even on fork PRs. This is the key advantage over `pull_request`. Exception: if `slash_command:` `events:` includes `pull_request`, PR-body commands *are* subject. Narrow `events:` to avoid. |
| Bot/Copilot events | Comments via `GITHUB_TOKEN` **do not** trigger. Comments via GitHub App tokens or PATs **do**. |
| Sanitize payload? | **Yes, always** in pre-agent steps. Comment/body text is user-controlled; use `steps.sanitized.outputs.text`, never raw `${{ github.event.comment.body }}`. Acceptable to handle unsanitized payload within the agent job (sandboxed), coupled with proper `safe-outputs`. |
| Safe-outputs | Minimum required for the command's purpose. Audit against `on.roles:` — if `all`, every safe-output is reachable by anyone who can comment. |

---

<nav>

[← Previous: `push`](push.md) | [Table of Contents](../README.md) | [Next: `discussion` →](discussion.md)

</nav>
