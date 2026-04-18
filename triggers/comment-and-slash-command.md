<nav>

[← Previous: `push` and `pull_request.synchronize`](push-and-synchronize.md) | [Table of Contents](../README.md) | [Next: `pull_request: types: [labeled]` and `label_command:` →](labeled-and-label-command.md)

</nav>

# `issue_comment` and `slash_command:`

Bundled because `slash_command:` subscribes to `issue_comment` (and friends) by default — the convenience trigger inherits every footgun of the underlying event, plus a few of its own.

---

## `issue_comment`

> 🛑 **A. The "no approval gate" loophole.** `issue_comment` does **NOT** trigger the "Approve and run workflows" gate, even when fired from a comment on a fork PR. A fork contributor can fire any `issue_comment`-listening workflow on their own PR with no maintainer interaction. This is the canonical reason `slash_command:` workflows have a much wider invocation surface than they appear — `slash_command:` subscribes to `issue_comment` by default.

> 🛑 **B. Issues vs. PRs naming trap.** Comments on PRs fire `issue_comment`, **NOT** `pull_request_review_comment` (which is a different event for inline review threads). Workflows authored as "an issue triager" inadvertently respond to comments on PRs from forks — even though the author never put `pull_request*` in `on:`. Distinguish via `github.event.issue.pull_request != null`.

> 🛑 **C. Edits to old comments fire NOW.** An attacker can edit a 6-month-old comment on a closed issue or PR, injecting `/command` or any payload — `issue_comment.edited` fires today against today's secrets, today's `permissions:`, today's `safe-outputs:` allowlist. The workflow has no concept of "this comment was created when our security model was different." [Tenet #10](../chapters/tenets.md)'s worst-case invocation rate must include the attacker reaching back through the entire comment history of the repo.

---

## `slash_command:` (gh-aw convenience trigger)

> 🛑 **`slash_command:` is the trigger most users mentally model as "fires only when someone with the right role types `/my-command`" — which is wrong by an order of magnitude or more, and the framing leads directly to the concurrency catastrophe.** What `slash_command:` actually does is subscribe to a *broad set of underlying events* (every issue comment, every PR review comment, every issue/PR open/edit, every discussion comment) and then filter at activation time. **Every one of those events spawns a workflow run** that consumes a runner slot and a brief execution context — only the `/command` filter and the `roles:` filter cause an early abort. Most workflow authors never see this because the runs abort fast and the UI lists "Skipped" — but they **did run**, they **did consume the runner**, and they **did affect concurrency groups**.
>
> *The concurrency catastrophe* (cross-reference [Concurrency and Race Conditions](../chapters/concurrency-and-races.md)): if the slash-command workflow declares `concurrency.group: my-cmd-${{ github.event.issue.number }}` with `cancel-in-progress: true`, then a benign drive-by comment by any read-role user on the same issue **cancels the in-flight `/command` run that a maintainer just authorized**. The maintainer sees their command silently fail mid-execution because Mallory typed "+1" thirty seconds later. The reverse failure mode is also live: without `cancel-in-progress`, every drive-by comment queues a (skip-fast) run behind the real one, and on busy issues the runner pool can saturate. There is no group key that gets this right — the broad subscription and the activation-time filter are fundamentally at odds with concurrency keyed on the conversation. **And `roles:` checks the actor of the comment that triggered the underlying event, not the actor of the original `/command` comment** — so a `.edited` event from anyone editing any old comment on the issue can fire the workflow under the *editor's* identity even if the editor never typed the slash-command.
>
> *Idempotency requirement:* because concurrency groups cannot be used safely with `slash_command:`, slash-command workflows **MUST be implemented to be idempotent** — by implementing the same concurrency checks or locking in code that schedule-jobs-that-poll must employ. Treat every `/command` activation as if the same `/command` might already be running for the same target: check before acting, claim a lock (e.g., via a label, an issue-comment marker, a check-run with a unique name, or external state), no-op if the work is already in progress or already done, and release/clean-up on completion. The discipline is the same as for scheduled pollers, and the failure modes when you skip it are the same: duplicate comments, duplicate labels, duplicate downstream effects, and races that leave the target in an indeterminate state.

---

<nav>

[← Previous: `push` and `pull_request.synchronize`](push-and-synchronize.md) | [Table of Contents](../README.md) | [Next: `pull_request: types: [labeled]` and `label_command:` →](labeled-and-label-command.md)

</nav>
