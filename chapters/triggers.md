---
title: "Triggers"
---

[← Previous: Tenets](tenets.md) | [Table of Contents](../README.md) | [Next: workflow_dispatch →](../triggers/workflow-dispatch.md)

# Triggers

Each page in this section covers one common trigger: when to use it, cross-cutting concern profile, and headline pitfall. The triggers are grouped by recommended usage level — **start at the top** when choosing a trigger for a new workflow. Within each group, triggers follow the lifecycle of activity on an issue or pull request.

Additional triggers not covered here (e.g., `branch_protection_rule`, `check_run`, `check_suite`, `create`/`delete`, `deployment`/`deployment_status`, `fork`, `gollum`, `label`, `member`, `page_build`, `project*`, `public`, `registry_package`, `repository_dispatch`, `status`, `watch`) either have low headline risk or are irrelevant to agentic workflows in this repo's context. They appear in [Appendix A: Trigger-by-Trigger Risk Profile](../appendices/trigger-risk-profile.md).

---

## Guidance key

| | Guidance | Meaning |
|---|---|---|
| ✅ | **Recommended** | Well-understood, safe defaults, straightforward concurrency story. |
| ⚠️ | **Use with caution** | Legitimate uses exist but the trigger has sharp edges that require deliberate configuration. |
| ☢️ | **Use with extreme caution** | Runs with full secrets and/or creates undeclared trust boundaries; the failure mode is repo compromise. |
| ⛔ | **Avoid** | Causes the "Approve and run workflows" button or other structural problems; prefer the recommended alternatives listed on the page. |

## ✅ Recommended

| Trigger | Headline |
|---|---|
| [`workflow_dispatch`](../triggers/workflow-dispatch.md) | Auto-paired with most triggers; manual escape hatch for debugging and ad-hoc runs. Any write-role user can fire against any branch. |
| [`schedule`](../triggers/schedule.md) | Preferred alternative to `pull_request` and `pull_request_target`. Focus on concurrency and idempotency. Be mindful of integrity filter, minimum `safe-outputs`, and avoiding unexpected escalation of privilege. |
| [`labeled` / `label_command:`](../triggers/labeled-and-label-command.md) | Human-in-the-loop gate via triage+ label application; preferred for on-demand operations with simple concurrency/idempotency scenarios. |
| [`issues`](../triggers/issue.md) | Best trigger for immediate, community-facing issue workflows with `safe-outputs` that do not require a human-in-the-loop gate. |
| [`release`](../triggers/release.md) | Post-release automation — release notes, follow-up issues, downstream notifications. Trusted trigger (write+). |
| [`milestone`](../triggers/milestone.md) | Release management automation on milestone close. Release notes, follow-up issues, downstream notifications. Trusted trigger (write+). |

## ⚠️ Use with caution

| Trigger | Headline |
|---|---|
| [`push`](../triggers/push.md) | Post-merge automation on push to `main`. Always use explicit `branches:` filters — bare `on: push` fires on every branch and is the trigger most likely to cause silent cost blowout. |
| [`issue_comment` / `slash_command:`](../triggers/comment-and-slash-command.md) | On-demand operations via `/slash_command` comments. Preferred over `pull_request` or `pull_request_target`, but the broad underlying event subscription causes concurrency, idempotency, spamming, and escalation of privilege pitfalls. Recommended alternative: `schedule`. |
| [`pull_request_review`](../triggers/pull-request-review.md) | Post-approval automation by filtering to approved event from users with write+ roles. Beware of concurrency/idempotency, broad underlying event subscription, and potential for privilege escalation. Recommended alternative: `schedule` or `labeled`/`label_command:`. |
| [`pull_request_review_comment`](../triggers/pull-request-review-comment.md) | Immediate triggering from inline code-review comments (many triggers per pull request review). Beware of concurrency, idempotency, noisy underlying trigger, and privilege escalation risks. Recommended alternatives: `schedule` or `labeled`/`label_command:`. |
| [`discussion`](../triggers/discussion.md) | Community Q&A and discussion routing. Most-open untrusted-input surface with no approval gate and lower visibility than issues. Prefer `schedule` for periodic processing if immediate triggering is not needed. |
| [`discussion_comment`](../triggers/discussion-comment.md) | Community Q&A and discussion routing. Most-open untrusted-input surface with no approval gate and lower visibility than issues. Prefer `schedule` for periodic processing if immediate triggering is not needed. |

## ☢️ Use with extreme caution

| Trigger | Headline |
|---|---|
| [`workflow_call`](../triggers/workflow-call.md) | Reusable workflow libraries. Undeclared trust boundary — callee inherits caller's permissions and secrets. Pin by SHA, not branch. `secrets: inherit` hands every secret to the callee. |
| [`workflow_run`](../triggers/workflow-run.md) | Post-CI actions with full secrets. `pull_request_target`'s quieter sibling — launders untrusted fork artifacts into a privileged context with no approval gate and no UI signal. Recommended alternatives: `schedule` or `labeled`/`label_command:`. |

## ⛔ Avoid

| Trigger | Headline |
|---|---|
| [`pull_request`](../triggers/pull-request.md) | Causes the "Approve and run workflows" button for all external contributor PRs (including Copilot). Can be safely used, even with fork PRs, but the button's existence creates alert fatigue that undermines the overall security model. Has the same concurrency/idempotency challenges as the recommended alternatives: `schedule`, `labeled`/`label_command:`. |
| [`pull_request_target`](../triggers/pull-request-target.md) | The trigger most likely to get a repo pwned. Runs on base ref with full secrets and write token, but often used to consume the PR's untrusted HEAD input. Causes the "Approve and run workflows" button for all external contributor PRs (including Copilot). Can be safely used when extreme care is taken, but the button's existence creates alert fatigue that undermines the security model. Has the same concurrency/idempotency challenges as the recommended alternatives: `schedule` and `labeled`/`label_command:`. |

> **No trigger is free of headline pitfalls.** Each page assumes you've internalized the [Tenets](tenets.md) — the pitfall framing is what they look like in production when a tenet is silently violated.

---

[← Previous: Tenets](tenets.md) | [Table of Contents](../README.md) | [Next: workflow_dispatch →](../triggers/workflow-dispatch.md)
