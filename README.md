# GitHub Actions & gh-aw Triggers — Field Guide

A working reference for every trigger usable in traditional GitHub Actions and in **GitHub Agentic Workflows (`gh-aw`)**, with emphasis on the surprises: apparent-vs-actual triggering, the *Approve and run workflows* gate, fork behavior, secrets exposure, concurrency-induced races, role-based authorization, and the read-only-contributor write surface.

The guide is structured around two complements:

- **[Tenets](chapters/tenets.md)** — the design principles every workflow must satisfy. *Positive framing: what we're trying to achieve.*
- **[Triggers](chapters/triggers.md)** — when to use each trigger, cross-cutting concern profiles, and headline pitfalls. *Negative framing: what to avoid.*

Pages can be read linearly via the `← Prev | TOC | Next →` navigation at the top and bottom of every page, or skipped to directly from the lists below.

---

## Chapters

1. [Tenets](chapters/tenets.md) — the 14 design principles every workflow must satisfy.
2. [Triggers](chapters/triggers.md) — index of per-trigger scenarios, profiles, and headline pitfalls (the per-trigger pages are listed below in lifecycle order).
3. [The Two Categories of Triggers](chapters/trigger-categories.md) — standard GitHub events vs. gh-aw convenience/virtual triggers; what every "trigger" actually compiles to.
4. [The "Apparent vs. Actual" Trigger Surface](chapters/apparent-vs-actual.md) — why "skipped" runs are not free.
5. [The "Approve and run workflows" Gate](chapters/approve-and-run-gate.md) — the gate is dangerous, not protective.
6. [Operating Within a Fork](chapters/operating-in-a-fork.md) — what fires when you operate inside your own fork, and the `if: workflow_dispatch || not-a-fork` guard.
7. [Concurrency and Race Conditions](chapters/concurrency-and-races.md) — the non-matching-cancels-matching pathology, the pre-cancellation race.
8. [Authorization, Roles, and Read-Only Contributors](chapters/authorization-and-roles.md) — `on.roles:` defaults, the read-only / fork-contributor capability matrix.

## Trigger Pages (lifecycle order)

1. [`issues`](triggers/issue.md) ✅ — `issues.edited` is a privilege-escalation amplifier.
2. [`pull_request`](triggers/pull-request.md) ⛔ — `closed` ≠ `merged`; the `reopened` time-bomb; the approval-gate problem.
3. [`pull_request_target`](triggers/pull-request-target.md) ⛔ — the trigger most likely to get a repo pwned.
4. [`push`](triggers/push.md) ⚠️ — overbroad branch subscription, per-push stacking, force-push semantics.
5. [`issue_comment` and `slash_command:`](triggers/comment-and-slash-command.md) ⚠️ — no approval gate, broad subscriptions, the `.edited` time-bomb, the slash-command concurrency catastrophe.
6. [`pull_request: types: [labeled]` and `label_command:`](triggers/labeled-and-label-command.md) ✅ — labels as one-shot RPC, audit-trail noise, idempotency requirement.
7. [`pull_request_review`](triggers/pull-request-review.md) ⚠️ — "review submitted" ≠ "approved," and the multi-trigger concurrency twist.
8. [`pull_request_review_comment`](triggers/pull-request-review-comment.md) ⚠️ — the trigger nobody remembers exists.
9. [`schedule`](triggers/schedule.md) ✅ — the best concurrency story; silent runtime cost growth and soft cron.
10. [`workflow_dispatch`](triggers/workflow-dispatch.md) ✅ — manual escape hatch that any write-role user can fire against any branch's YAML.
11. [`discussion`](triggers/discussion.md) ⚠️ — the most-open untrusted-input surface in the GitHub model.
12. [`discussion_comment`](triggers/discussion-comment.md) ⚠️ — comment-on-your-own-discussion privilege escalation.
13. [`release`](triggers/release.md) ✅ — post-release automation, artifact publishing, follow-up issues.
14. [`milestone`](triggers/milestone.md) ✅ — release management; cascade when deleting.
15. [`workflow_call`](triggers/workflow-call.md) ☢️ — undeclared trust boundary; `secrets: inherit`.
16. [`workflow_run`](triggers/workflow-run.md) ☢️ — `pull_request_target`'s quieter sibling.

## Appendices

- [Appendix A: Trigger-by-Trigger Risk Profile](appendices/trigger-risk-profile.md) — the all-triggers reference table including ones with no standalone pitfall page.
- [Appendix B: Footnotes](appendices/footnotes.md) — source citations for every claim in the guide.

---

## Suggested reading order

If you're new to gh-aw and to the surprises this guide catalogues, read it linearly via the Prev/Next links — start with [Tenets](chapters/tenets.md) and follow the chain. Returning readers can jump straight to a trigger page or to a deep-dive chapter.
