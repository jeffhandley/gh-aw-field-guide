<nav>

<a href="tenets.md">← Previous: Tenets</a> | <a href="../README.md">Table of Contents</a> | <a href="../triggers/issue.md">Next: `issues` →</a>

</nav>

# Triggers

Each page in this section covers one common trigger: when to use it, cross-cutting concern profile, and headline pitfall. The pages are ordered to follow the **lifecycle of activity on an issue or pull request**, then move outward to schedule/dispatch/discussion/release/cascade triggers, and finally to the cross-workflow triggers (`workflow_call`, `workflow_run`).

Additional triggers not covered here (e.g., `branch_protection_rule`, `check_run`, `check_suite`, `create`/`delete`, `deployment`/`deployment_status`, `fork`, `gollum`, `label`, `member`, `page_build`, `project*`, `public`, `registry_package`, `repository_dispatch`, `status`, `watch`) either have low headline risk or are irrelevant to agentic workflows in this repo's context. They appear in [Appendix A: Trigger-by-Trigger Risk Profile](../appendices/trigger-risk-profile.md).

---

## Guidance key

| | Guidance | Meaning |
|---|---|---|
| ✅ | **Recommended** | Well-understood, safe defaults, straightforward concurrency story. |
| ⚠️ | **Use with caution** | Legitimate uses exist but the trigger has sharp edges that require deliberate configuration. |
| ☢️ | **Use with extreme caution** | Runs with full secrets and/or creates undeclared trust boundaries; the failure mode is repo compromise. |
| ⛔ | **Avoid** | Causes the "Approve and run workflows" button or other structural problems; prefer the recommended alternatives listed on the page. |

## Common triggers

| Trigger | | Headline |
|---|---|---|
| [`issues`](../triggers/issue.md) | ✅ | `.edited` is a privilege-escalation amplifier. |
| [`pull_request`](../triggers/pull-request.md) | ⛔ | `closed` ≠ `merged`; the `reopened` time-bomb; causes the approval-gate button. |
| [`pull_request_target`](../triggers/pull-request-target.md) | ⛔ | The trigger most likely to get a repo pwned. |
| [`push`](../triggers/push.md) | ⚠️ | Overbroad branch subscription; per-push stacking; force-push semantics. |
| [`issue_comment` / `slash_command:`](../triggers/comment-and-slash-command.md) | ⚠️ | No approval gate; broad subscriptions; the `.edited` time-bomb; concurrency catastrophe. |
| [`discussion`](../triggers/discussion.md) | ⚠️ | Most-open untrusted-input surface; no approval gate; low visibility. |
| [`discussion_comment`](../triggers/discussion-comment.md) | ⚠️ | Comment-on-your-own-discussion privilege escalation. |
| [`labeled` / `label_command:`](../triggers/labeled-and-label-command.md) | ✅ | Labels as one-shot RPC; audit-trail noise; idempotency requirement. |
| [`pull_request_review`](../triggers/pull-request-review.md) | ⚠️ | "Review submitted" ≠ "approved"; multi-trigger concurrency twist. |
| [`pull_request_review_comment`](../triggers/pull-request-review-comment.md) | ⚠️ | The trigger nobody remembers exists. |
| [`schedule`](../triggers/schedule.md) | ✅ | Best concurrency story; silent runtime cost growth; soft cron. |
| [`workflow_dispatch`](../triggers/workflow-dispatch.md) | ✅ | Manual escape hatch; any write-role user can fire against any branch. |
| [`release`](../triggers/release.md) | ✅ | Post-release automation; artifact publishing; follow-up issues. |
| [`milestone`](../triggers/milestone.md) | ✅ | Release management; cascade when deleting. |
| [`workflow_call`](../triggers/workflow-call.md) | ☢️ | Undeclared trust boundary; `secrets: inherit`. |
| [`workflow_run`](../triggers/workflow-run.md) | ☢️ | `pull_request_target`'s quieter sibling. |

> **No trigger is free of headline pitfalls.** Each page assumes you've internalized the [Tenets](tenets.md) — the pitfall framing is what they look like in production when a tenet is silently violated.

---

<nav>

<a href="tenets.md">← Previous: Tenets</a> | <a href="../README.md">Table of Contents</a> | <a href="../triggers/issue.md">Next: `issues` →</a>

</nav>
