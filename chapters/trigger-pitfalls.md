<nav>

[← Previous: Tenets](tenets.md) | [Table of Contents](../README.md) | [Next: Issue lifecycle →](../triggers/issue.md)

</nav>

# Trigger Pitfalls

Each page in this section names the worst-case footgun(s) for one trigger — the mistakes most likely to bite you in production. The pages are ordered to follow the **lifecycle of activity on an issue or pull request**, then move outward to schedule/dispatch/discussion/cascade triggers, and finally to the cross-workflow triggers (`workflow_call`, `workflow_run`).

The full per-trigger reference (events, activity types, fork behavior, role gates) — including triggers with low headline risk that are not given a standalone page here — lives in [Appendix A: Trigger-by-Trigger Risk Profile](../appendices/trigger-risk-profile.md).

> Triggers not given a standalone page (e.g., `branch_protection_rule`, `check_run`, `check_suite`, `create`/`delete`, `deployment`/`deployment_status`, `fork`, `gollum`, `label`, `member`, `page_build`, `project*`, `public`, `registry_package`, `release`, `repository_dispatch`, `status`, `watch`) either have low headline risk or are irrelevant to agentic workflows in this repo's context. They appear in [Appendix A](../appendices/trigger-risk-profile.md).

---

## Pages, in lifecycle order

1. [`issues`](../triggers/issue.md) — open / edit / close, and the `.edited` privilege-escalation amplifier.
2. [`pull_request`](../triggers/pull-request.md) — `closed` ≠ `merged`, and the `reopened` time-bomb.
3. [`pull_request_target`](../triggers/pull-request-target.md) ⚠️ — the trigger most likely to get a repo pwned.
4. [`push` and `pull_request.synchronize`](../triggers/push-and-synchronize.md) — overbroad branch subscription, per-push stacking, force-push semantics.
5. [`issue_comment` and `slash_command:`](../triggers/comment-and-slash-command.md) — no approval gate, broad subscriptions, the `.edited` time-bomb, the slash-command concurrency catastrophe.
6. [`pull_request: types: [labeled]` and `label_command:`](../triggers/labeled-and-label-command.md) — labels as one-shot RPC, audit-trail noise, idempotency requirement.
7. [`pull_request_review`](../triggers/pull-request-review.md) — "review submitted" ≠ "approved," and the multi-trigger concurrency twist.
8. [`schedule`](../triggers/schedule.md) — silent runtime cost growth, soft cron, and the polling-vs-eventing structural problem.
9. [`workflow_dispatch`](../triggers/workflow-dispatch.md) — manual escape hatch that any write-role user can fire against any branch's YAML.
10. [`discussion`](../triggers/discussion.md) — the most-open untrusted-input surface in the GitHub model, with no approval gate.
11. [`discussion_comment`](../triggers/discussion-comment.md) — comment-on-your-own-discussion privilege escalation.
12. [`pull_request_review_comment`](../triggers/pull-request-review-comment.md) — the trigger nobody remembers exists, ingesting attacker-positioned prose against specific code lines.
13. [`milestone`](../triggers/milestone.md) — cascade trigger; deleting one milestone fires N events.
14. [`workflow_call`](../triggers/workflow-call.md) — undeclared trust boundary; `secrets: inherit` hands every secret to the callee.
15. [`workflow_run`](../triggers/workflow-run.md) ⚠️ — `pull_request_target`'s quieter sibling: launders fork artifacts into a privileged context.

> **No trigger is free of headline pitfalls.** Each page assumes you've internalized the [Tenets](tenets.md) — the pitfall framing is what they look like in production when a tenet is silently violated.

---

<nav>

[← Previous: Tenets](tenets.md) | [Table of Contents](../README.md) | [Next: Issue lifecycle →](../triggers/issue.md)

</nav>
