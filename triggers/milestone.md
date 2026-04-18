<nav>

[← Previous: `pull_request_review_comment`](pull-request-review-comment.md) | [Table of Contents](../README.md) | [Next: `workflow_call` →](workflow-call.md)

</nav>

# `milestone`

> 🛑 **`milestone.closed` is a cascade trigger that workflows underestimate.** Closing a milestone is a single admin action that does NOT itself fire issue events — but **DELETING** a milestone removes it from all assigned issues/PRs, firing `issues.demilestoned` (and `pull_request.demilestoned`) for every one of them in rapid succession. A workflow that listens on `*.demilestoned` to react (move from a project, change labels, post a comment) will fire N times in seconds when a milestone with N items is deleted. Combined with `cancel-in-progress: true` concurrency, only the last one runs and earlier reactions are silently lost; without it, runner minutes balloon. Same naming-collision footgun as [`label`](labeled-and-label-command.md): `on: milestone` is **not** `on: issues: types: [milestoned]`.

---

<nav>

[← Previous: `pull_request_review_comment`](pull-request-review-comment.md) | [Table of Contents](../README.md) | [Next: `workflow_call` →](workflow-call.md)

</nav>
