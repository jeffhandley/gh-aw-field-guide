# GitHub Actions & gh-aw Triggers — Field Guide

A working reference for every trigger usable in traditional GitHub Actions and in **GitHub Agentic Workflows (`gh-aw`)**, with emphasis on the surprises: apparent-vs-actual triggering, the *Approve and run workflows* gate, fork behavior, secrets exposure, concurrency-induced races, role-based authorization, and the read-only-contributor write surface.

- **[Tenets](chapters/tenets.md)** — the design principles every workflow must satisfy.
- **[Triggers](chapters/triggers.md)** — when to use each trigger, cross-cutting concern profiles, and headline pitfalls.

---

## More Thoughts

1. [The Two Categories of Triggers](chapters/trigger-categories.md) — standard GitHub events vs. gh-aw convenience/virtual triggers; what every "trigger" actually compiles to.
2. [The "Apparent vs. Actual" Trigger Surface](chapters/apparent-vs-actual.md) — why "skipped" runs are not free.
3. [The "Approve and run workflows" Gate](chapters/approve-and-run-gate.md) — the gate is dangerous, not protective.
4. [Operating Within a Fork](chapters/operating-in-a-fork.md) — what fires when you operate inside your own fork, and the `if: workflow_dispatch || not-a-fork` guard.
5. [Concurrency and Race Conditions](chapters/concurrency-and-races.md) — the non-matching-cancels-matching pathology, the pre-cancellation race.
6. [Authorization, Roles, and Read-Only Contributors](chapters/authorization-and-roles.md) — `on.roles:` defaults, the read-only / fork-contributor capability matrix.

## Appendices

- [Appendix A: Trigger-by-Trigger Risk Profile](appendices/trigger-risk-profile.md) — the all-triggers reference table including ones with no standalone page.
- [Appendix B: Footnotes](appendices/footnotes.md) — source citations for every claim in the guide.
