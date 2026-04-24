# GitHub Agentic Workflows — Field Guide

## Tenets

The design principles every workflow must consider and strive to satisfy. These tenets inform the usage guidelines and recommendations throughout this guide.

[Read the Tenets →](chapters/tenets.md)

## Triggers

A reference for triggers used in [traditional GitHub Workflows](https://docs.github.com/actions/writing-workflows) and in [GitHub Agentic Workflows (`gh-aw`)](https://gh.io/gh-aw), covering how security and other tenets apply to each trigger. Scenarios, high-level guidance, and notable pitfalls.

[Study the Common Triggers →](chapters/triggers.md)

---

## More Thoughts

1. [The "Approve and run workflows" Gate](chapters/approve-and-run-gate.md) — the gate is dangerous, not protective.
1. [The "Apparent vs. Actual" Trigger Surface](chapters/apparent-vs-actual.md) — why "skipped" runs are not free.
1. [Operating Within a Fork](chapters/operating-in-a-fork.md) — what fires when you operate inside your own fork, and the `if: workflow_dispatch || not-a-fork` guard.
1. [Concurrency and Race Conditions](chapters/concurrency-and-races.md) — the non-matching-cancels-matching pathology, the pre-cancellation race.
1. [Authorization, Roles, and Read-Only Contributors](chapters/authorization-and-roles.md) — `on.roles:` defaults, the read-only / fork-contributor capability matrix.

## Appendices

- [Trigger-by-Trigger Risk Profile](appendices/trigger-risk-profile.md) — the all-triggers reference table including ones with no standalone page.
