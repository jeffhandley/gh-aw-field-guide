<nav>

[← Previous: Tenets](tenets.md) | [Table of Contents](../README.md) | [Next: `issues` →](../triggers/issue.md)

</nav>

# Triggers

Each page in this section covers one trigger: when to use it, cross-cutting concern profile, and headline pitfall. The pages are ordered to follow the **lifecycle of activity on an issue or pull request**, then move outward to schedule/dispatch/discussion/release/cascade triggers, and finally to the cross-workflow triggers (`workflow_call`, `workflow_run`).

The full per-trigger reference (events, activity types, fork behavior, role gates) — including triggers with low headline risk that are not given a standalone page here — lives in [Appendix A: Trigger-by-Trigger Risk Profile](../appendices/trigger-risk-profile.md).

> Triggers not given a standalone page (e.g., `branch_protection_rule`, `check_run`, `check_suite`, `create`/`delete`, `deployment`/`deployment_status`, `fork`, `gollum`, `label`, `member`, `page_build`, `project*`, `public`, `registry_package`, `repository_dispatch`, `status`, `watch`) either have low headline risk or are irrelevant to agentic workflows in this repo's context. They appear in [Appendix A](../appendices/trigger-risk-profile.md).

---

## Guidance key

- ✅ **Recommended** — well-understood, safe defaults, straightforward concurrency story.
- ⚠️ **Use with caution** — legitimate uses exist but the trigger has sharp edges that require deliberate configuration.
- ⛔ **Avoid** — causes the "Approve and run workflows" button or other structural problems; prefer the recommended alternatives listed on the page.
- ☢️ **Use with extreme caution** — runs with full secrets and/or creates undeclared trust boundaries; the failure mode is repo compromise.

## Pages, in lifecycle order

1. [`issues`](../triggers/issue.md) ✅ — open / edit / close, and the `.edited` privilege-escalation amplifier.
2. [`pull_request`](../triggers/pull-request.md) ⛔ — `closed` ≠ `merged`, the `reopened` time-bomb, and the approval-gate problem.
3. [`pull_request_target`](../triggers/pull-request-target.md) ⛔ — the trigger most likely to get a repo pwned.
4. [`push`](../triggers/push.md) ⚠️ — overbroad branch subscription, per-push stacking, force-push semantics.
5. [`issue_comment` and `slash_command:`](../triggers/comment-and-slash-command.md) ⚠️ — no approval gate, broad subscriptions, the `.edited` time-bomb, the slash-command concurrency catastrophe.
6. [`pull_request: types: [labeled]` and `label_command:`](../triggers/labeled-and-label-command.md) ✅ — labels as one-shot RPC, audit-trail noise, idempotency requirement.
7. [`pull_request_review`](../triggers/pull-request-review.md) ⚠️ — "review submitted" ≠ "approved," and the multi-trigger concurrency twist.
8. [`pull_request_review_comment`](../triggers/pull-request-review-comment.md) ⚠️ — the trigger nobody remembers exists.
9. [`schedule`](../triggers/schedule.md) ✅ — the best concurrency story; silent runtime cost growth and soft cron.
10. [`workflow_dispatch`](../triggers/workflow-dispatch.md) ✅ — manual escape hatch that any write-role user can fire against any branch's YAML.
11. [`discussion`](../triggers/discussion.md) ⚠️ — the most-open untrusted-input surface in the GitHub model.
12. [`discussion_comment`](../triggers/discussion-comment.md) ⚠️ — comment-on-your-own-discussion privilege escalation.
13. [`release`](../triggers/release.md) ✅ — post-release automation, artifact publishing, follow-up issues.
14. [`milestone`](../triggers/milestone.md) ✅ — release management; cascade when deleting.
15. [`workflow_call`](../triggers/workflow-call.md) ☢️ — undeclared trust boundary; `secrets: inherit`.
16. [`workflow_run`](../triggers/workflow-run.md) ☢️ — `pull_request_target`'s quieter sibling.

> **No trigger is free of headline pitfalls.** Each page assumes you've internalized the [Tenets](tenets.md) — the pitfall framing is what they look like in production when a tenet is silently violated.

---

<nav>

[← Previous: Tenets](tenets.md) | [Table of Contents](../README.md) | [Next: `issues` →](../triggers/issue.md)

</nav>
