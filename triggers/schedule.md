<nav>

[← Previous: `pull_request_review_comment`](pull-request-review-comment.md) | [Table of Contents](../README.md) | [Next: `workflow_dispatch` →](workflow-dispatch.md)

</nav>

# `schedule`

### ✅ Recommended

> 🛑 **`schedule` is the trigger that keeps running quietly in the background without relying on any user interaction at all.** These workflows are forced to re-derive context that an event would have handed them, often burning tokens on a fuzzy clock against stale assumptions. Scheduled agentic workflows run a cron against today's default branch but were designed around yesterday's repo. A scheduled "triage open issues" workflow written when there were 20 open issues may still be running years later against 2,000 open issues, with per-run cost now 100× what was tested. There is no PR to review, no comment to acknowledge, no UI indicator that the workflow even ran — it just consumes runner minutes and Copilot tokens indefinitely. The "schedule" is also softer than it reads: GitHub runs the cron **when a runner becomes available**, not on a fixed wall clock — heavy load delays runs by minutes-to-hours, and during peak hours your every-15-minute job may run every 25.
>
> The deeper structural problem: **`schedule` is the wrong tool for work that should react to events.** Where a real trigger hands the workflow a payload that says exactly what changed, a scheduled run starts blind and must **poll** to discover what work might exist, track its **own state** to know what's already been done, and implement some form of **optimistic concurrency or locking** so two overlapping cron firings don't double-process the same item. Every one of those is a distributed-systems problem the workflow author now has to solve in YAML and shell — and most don't, so scheduled agents quietly post duplicate comments, file duplicate issues, or apply duplicate labels every time the cron laps the prior run.

---

## Scenarios

- Periodic housekeeping — stale issue/PR sweeps, label audits, project-board syncing
- Polling-based PR operations that avoid the "Approve and run workflows" button — the preferred alternative to `pull_request` and `pull_request_target` triggers
- Recurring reports — CI health, security audit, dependency freshness
- **Best concurrency story of any trigger** — no event spamming, no user-triggered invocations, no race with drive-by comments

## Profile

| Dimension | Recommendation |
|---|---|
| `on.roles:` | N/A — no actor involved. The workflow runs as the default branch's workflow file. |
| Activity types | N/A. Use gh-aw schedule shorthands (`daily`, `weekly on monday`, `every 10 minutes`, `daily around 14:00`, `daily between 9:00 and 17:00`) to spread load. Raw cron is also supported. **Strongly recommended to pair with `workflow_dispatch`** to allow manual triggers off-schedule (for debugging, catch-up runs, or ad-hoc invocation). |
| Concurrency | `${{ github.workflow }}` (no issue/PR number to key on). `cancel-in-progress` depends on whether overlapping cron firings should stack or replace — typically `true` for idempotent pollers. Concurrency is substantially more straightforward than event-driven triggers because the invocation rate is controlled and predictable. |
| Idempotency | **Required.** Two overlapping cron firings must not double-process. Track state via structured `<!-- comments -->` inside the issue/PR description or a comment, labels, or external state — but take note that cancellation and retries need to be incorporated into the state-tracking design. The workflow starts blind and must poll to discover work — design for exactly-once processing. |
| Fork posture | GitHub disables `schedule` triggers on forks by default until the fork owner re-enables them in the Actions tab. Apply `if: ${{ github.event_name == 'workflow_dispatch' \|\| !github.event.repository.fork }}` as defense-in-depth. |
| Approval gate | Not subject to the "Approve and run workflows" button. |
| Bot/Copilot events | N/A — schedule is time-driven, not event-driven. |
| Sanitize payload? | No event payload to sanitize. However, the data the workflow *polls* (issue bodies, PR titles, comments) is still user-controlled — sanitize anything ingested from the GitHub API before passing to the agent. Acceptable to handle unsanitized polled data within the agent job (sandboxed), coupled with proper `safe-outputs`. |
| Safe-outputs | Depends on workflow purpose. `add-labels`, `add-comment`, `create-issue`, `update-issue`, `create-pull-request`, `push-to-pull-request-branch` are all common for housekeeping pollers. Cost consciousness is critical — a scheduled "triage open issues" workflow written for 20 issues may still run years later against 2,000 issues at 100× the original per-run cost. |

---

<nav>

[← Previous: `pull_request_review_comment`](pull-request-review-comment.md) | [Table of Contents](../README.md) | [Next: `workflow_dispatch` →](workflow-dispatch.md)

</nav>
