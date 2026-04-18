<nav>

[← Previous: `pull_request: types: [labeled]` and `label_command:`](labeled-and-label-command.md) | [Table of Contents](../README.md) | [Next: `schedule` →](schedule.md)

</nav>

# `pull_request_review`

> 🛑 **"A review was submitted" is not "the PR was approved" — but workflows conflate the two.** The event fires for any review type, including `COMMENT`-type reviews that any read-role user (or any drive-by GitHub account) can submit. Workflows that auto-merge "after N reviews submitted," post "ready to land" comments, or kick off deployment on review-submitted fire for non-approving comments too — including comments authored by accounts with no write access at all. Even gating correctly on `event.review.state == 'approved'` doesn't suffice because actor authorization is independent of review state. Review bodies are unstructured prose authored by anyone with read access — an under-monitored prompt-injection vector that workflows ingest as if it came from a trusted reviewer. `pull_request_review.edited` fires today against today's secrets when an old review is edited (same time-bomb shape as [`issue_comment.edited`](comment-and-slash-command.md)).
>
> *Concurrency twist:* `pull_request_review` is rarely the only trigger on a PR-handling workflow; it usually shares a workflow file with `pull_request` and `issue_comment`. If they share `concurrency.group: pr-${{ github.event.pull_request.number || github.event.issue.number }}` with `cancel-in-progress: true`, a stray comment or a reviewer dismissing an old review can cancel a half-finished APPROVE-handling run — the activation passed, the agent is doing real work, then a benign comment kills it mid-flight, leaving the PR in an indeterminate state (label half-applied, status check half-posted). Without `cancel-in-progress`, two near-simultaneous reviewers stack runs and post duplicate comments or apply duplicate labels. There is no group key that gets this right for all three triggers in one workflow.

---

<nav>

[← Previous: `pull_request: types: [labeled]` and `label_command:`](labeled-and-label-command.md) | [Table of Contents](../README.md) | [Next: `schedule` →](schedule.md)

</nav>
