<nav>

[← Previous: `pull_request_review`](pull-request-review.md) | [Table of Contents](../README.md) | [Next: `workflow_dispatch` →](workflow-dispatch.md)

</nav>

# `schedule`

> 🛑 **`schedule` is the trigger that keeps running quietly in the background without relying on any user interaction at all.** These workflows are forced to re-derive context that an event would have handed them, often burning tokens on a fuzzy clock against stale assumptions. Scheduled agentic workflows run a cron against today's default branch but were designed around yesterday's repo. A scheduled "triage open issues" workflow written when there were 20 open issues may still be running years later against 2,000 open issues, with per-run cost now 100× what was tested. There is no PR to review, no comment to acknowledge, no UI indicator that the workflow even ran — it just consumes runner minutes and Copilot tokens indefinitely. The "schedule" is also softer than it reads: GitHub runs the cron **when a runner becomes available**, not on a fixed wall clock — heavy load delays runs by minutes-to-hours, and during peak hours your every-15-minute job may run every 25.
>
> The deeper structural problem: **`schedule` is the wrong tool for work that should react to events.** Where a real trigger hands the workflow a payload that says exactly what changed, a scheduled run starts blind and must **poll** to discover what work might exist, track its **own state** to know what's already been done, and implement some form of **optimistic concurrency or locking** so two overlapping cron firings don't double-process the same item. Every one of those is a distributed-systems problem the workflow author now has to solve in YAML and shell — and most don't, so scheduled agents quietly post duplicate comments, file duplicate issues, or apply duplicate labels every time the cron laps the prior run.

---

<nav>

[← Previous: `pull_request_review`](pull-request-review.md) | [Table of Contents](../README.md) | [Next: `workflow_dispatch` →](workflow-dispatch.md)

</nav>
