<nav>

[← Previous: `pull_request_target`](pull-request-target.md) | [Table of Contents](../README.md) | [Next: `issue_comment` and `slash_command:` →](comment-and-slash-command.md)

</nav>

# `push` and `pull_request.synchronize`

These two are bundled because they're the same underlying activity (a commit lands on a ref) viewed from two angles: `push` from the branch's angle, and `pull_request.synchronize` from a PR's angle. They share the same per-push fan-out problem and the same concurrency trap.

---

## `push` — overbroad branch subscription

> 🛑 **`push` is one of the highest-volume triggers in the repo and the easiest to subscribe to overbroadly.** A workflow declared as just `on: push` (no `branches:` filter) fires on every push to every branch by anyone with write access — including bot branches, dependency-update branches, codeflow branches, and short-lived feature branches that no one expected the workflow to care about. For an agentic workflow that consumes Copilot tokens per run, the cost blowout is silent until the bill arrives — this is the trigger most likely to turn "it's just a PoC" into a line-item on next month's invoice. The fix is always-explicit `branches:` filters and a narrow subscription that matches the actual workflow intent.
>
> *Concurrency twist:* rapid pushes to a feature branch (rebasing, force-pushing) stack up runs unless `concurrency.cancel-in-progress: true` is set — which in turn re-creates the race condition (see [Concurrency and Race Conditions](../chapters/concurrency-and-races.md)) where the canceled run was halfway through a write that the next push won't redo.

---

## `pull_request.synchronize` — per-push, not per-PR

> 🛑 **`synchronize` fires once per push, not once per PR.** A 50-commit force-push to a PR fires the workflow 50 times — wait, no: a 50-commit *single push* fires it **once** (the event is per push, not per commit). But a *force-push* to rewrite history is still a push, and a *contributor pushing 50 separate times in 30 minutes* is 50 events. Force-push specifically is indistinguishable to the workflow from a regular push — both fire `synchronize` with the new head SHA — so the workflow cannot tell whether the previous commits it already analyzed are still in the tree or have been rewritten away.
>
> The trade-off is unavoidable: enabling `concurrency.cancel-in-progress: true` solves the per-push stacking but creates the [non-matching-cancels-matching race](../chapters/concurrency-and-races.md) where in-progress useful work gets cancelled by a spam push (or by a sibling event sharing the same group). The cost-vs-correctness tradeoff is unavoidable; it must be made deliberately.

### Things that *do not* fire `synchronize` (despite reading like they should)

- **Draft ↔ ready-for-review conversions.** Marking a PR as ready fires `pull_request.ready_for_review`; converting back to draft fires `pull_request.converted_to_draft`. Neither fires `synchronize`. Workflows whose `on:` block uses default `types: [opened, synchronize, reopened]` will **not** run when a long-running draft is finally marked ready, even though "the PR just changed" intuitively feels like a sync.
- **Base-ref edits.** Changing the PR's base branch fires `pull_request.edited` (with `changes.base` populated), not `synchronize`. Workflows that rely on `synchronize` to re-run CI against the new base will silently skip.
- **Reviews, comments, label changes.** All have their own activity types on `pull_request` or fire entirely separate events (`pull_request_review`, `issue_comment`).
- **Pushes to the *base* branch** (e.g., someone merges to `main` while your PR targets `main`). This does not fire `synchronize` on your open PRs — it fires `push` on `main`. Your PR's CI won't re-run against the new base unless you push to your own branch (or close-and-reopen).

### Branch-protection interaction (often surprising)

Branch protection's "Dismiss stale pull request approvals when new commits are pushed" setting fires on the same head-SHA-changed event that drives `synchronize`. This is independent of your workflow — but it means a `synchronize` from any push (including a force-push that didn't actually change file contents, e.g., a rebase onto current `main`) silently invalidates every prior approval. A workflow that gates downstream behavior on "PR is approved" can therefore see the approval state flip back to "0 approvals" mid-run.

### Head-SHA invalidation

The PR head SHA changes on every push. Any artifact the workflow created (status check, comment, posted analysis) referencing the *previous* head SHA is now stale — and a follow-up `pull_request_target` workflow that checks out `head.sha` is now checking out **different code** than the prior run analyzed. There is no built-in mechanism to "carry forward" prior-run state across a `synchronize`; idempotency must be explicit.

---

<nav>

[← Previous: `pull_request_target`](pull-request-target.md) | [Table of Contents](../README.md) | [Next: `issue_comment` and `slash_command:` →](comment-and-slash-command.md)

</nav>
