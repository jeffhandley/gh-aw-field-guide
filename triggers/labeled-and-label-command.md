<nav>

[← Previous: `issue_comment` and `slash_command:`](comment-and-slash-command.md) | [Table of Contents](../README.md) | [Next: `pull_request_review` →](pull-request-review.md)

</nav>

# `pull_request: types: [labeled]` and `label_command:`

Two ways to use a label as a workflow signal — the underlying GitHub event (`pull_request.labeled` / `issues.labeled` / `discussion.labeled`) and the gh-aw convenience trigger that wraps them. They share most of the same pitfalls, with `label_command:` adding a one-shot-RPC twist.

> Don't confuse these with the `on: label` GitHub event, which fires on **repository-level label CRUD** (creating/editing/deleting label definitions in repo settings) — *not* on label *application*. That's a separate trigger and lives in [Appendix A](../appendices/trigger-risk-profile.md).

---

## `*.labeled` activity types (`issues`, `pull_request`, `discussion`)

> 🛑 **The `labeled` activity type fires under the labeller's identity, but the payload remains attacker-controlled.** A workflow gated on `github.event.sender` ("only act if a maintainer triggered me") can be tricked: a read-only user opens an issue with attacker prose, a triage-role maintainer applies the `bug` label (innocuous), and the agent now runs as if the maintainer authored the issue. The actor changed; the body, title, and comments did not. The same shape repeats for `assigned`, `milestoned`, `pinned`, etc. — any activity type a maintainer-role actor can perform inherits the issue/PR's existing attacker-controlled payload.

> 🛑 **Cascade fan-out.** Bulk label operations — applying a label across many issues at once via the GitHub UI's bulk-action menu, a script, or a "label everything matching X" workflow — fire one event per affected item in rapid succession. A workflow listening on `pull_request: types: [labeled]` will fire N times in seconds. With `cancel-in-progress: true` only the last one runs (silently dropping work for the earlier N-1 items); without it, runner minutes balloon. The same cascade shape applies to `*.demilestoned` when a milestone is deleted (see [`milestone`](milestone.md)) and to `*.unlabeled` when a label definition is deleted from the repo.

---

## `label_command:` (gh-aw convenience trigger)

> 🛑 **`label_command:` turns labels into one-shot RPC calls — and inherits the same broad-subscription concurrency catastrophe as `slash_command:`.** The trigger fires when any write-role contributor applies the configured label, runs the agentic workflow, and then **automatically removes the label**. The audit trail is preserved but is *subtle* — the issue/PR event stream shows `Octocat added the deploy label` immediately followed by `github-actions[bot] removed the deploy label`, which is easy to scroll past as a label-fix correction. For a `/deploy`-shaped command, this means a triage-role contributor (write access but typically not deploy-trusted) can fire the deploy by applying a label, and the only record of the command being issued lives in two adjacent timeline events that look like noise.
>
> The bigger structural problem is the same shape as [`slash_command:`](comment-and-slash-command.md): `label_command:` subscribes to *every* `labeled` event on the configured event types (`issues`, `pull_request`, `discussion`). The workflow runs — consuming a runner — on every label application by every write-role user, and only the name match causes early abort. Like `slash_command:`, this means **concurrency groups cannot safely be used** (a benign label change on the same target would cancel the in-flight command). Therefore `label_command:` workflows **MUST be implemented to be idempotent**: check before acting, claim a lock, no-op if the work is already in progress or already done. The same discipline scheduled pollers and `slash_command:` workflows must employ. Skipping that discipline produces the same failure modes: duplicate deploys, duplicate comments, duplicate downstream effects, and races that leave the target in an indeterminate state.

---

<nav>

[← Previous: `issue_comment` and `slash_command:`](comment-and-slash-command.md) | [Table of Contents](../README.md) | [Next: `pull_request_review` →](pull-request-review.md)

</nav>
