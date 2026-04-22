---
title: "milestone"
---

[← Previous: release](release.md) | [Table of Contents](../README.md) | [Next: push →](push.md)

# `milestone`

### ✅ Recommended

> 🛑 **`milestone.closed` is a cascade trigger that workflows underestimate.** Closing a milestone is a single admin action that does NOT itself fire issue events — but **DELETING** a milestone removes it from all assigned issues/PRs, firing `issues.demilestoned` (and `pull_request.demilestoned`) for every one of them in rapid succession. A workflow that listens on `*.demilestoned` to react (move from a project, change labels, post a comment) will fire N times in seconds when a milestone with N items is deleted. Combined with `cancel-in-progress: true` concurrency, only the last one runs and earlier reactions are silently lost; without it, runner minutes balloon. Same naming-collision footgun as [`label`](labeled-and-label-command.md): `on: milestone` is **not** `on: issues: types: [milestoned]`.

---

## Scenarios

- Release management automation on `milestone.closed` — generate release notes, file follow-up issues, update project boards
- Milestone-driven triage — react to `milestone.created` or `milestone.edited` to set up project structure

## Profile

| Dimension | Recommendation |
|---|---|
| `on.roles:` | N/A — milestone operations require write+ access. |
| Activity types | `[closed]` for release automation. `[created]` for setup automation. `deleted` is rarely useful as a trigger — prefer reacting to the cascade events below if you need per-item behavior. |
| Concurrency | `${{ github.workflow }}-${{ github.event.milestone.number }}`. `cancel-in-progress` depends on whether overlapping milestone operations should stack or replace. |
| Idempotency | **Required.** The workflow must be safe to repeat if the milestone is reopened/reclosed. |
| Fork posture | Apply `if: ${{ github.event_name == 'workflow_dispatch' \|\| !github.event.repository.fork }}` to prevent running within a user's fork. |
| Approval gate | Not subject to the "Approve and run workflows" button. |
| Bot/Copilot events | Milestone operations via `GITHUB_TOKEN` **do not** trigger. Operations via GitHub App tokens or PATs **do**. |
| Sanitize payload? | Milestone title and description are maintainer-controlled and generally trusted (write access required). Issue/PR bodies linked to the milestone remain user-controlled — sanitize those if ingested. Acceptable to handle unsanitized payload within the agent job (sandboxed), coupled with proper `safe-outputs`. |
| Safe-outputs | `create-issue`, `add-comment`, `add-labels`, `create-pull-request`, `update-issue`, `create-discussion` for release automation. `workflow_dispatch` via `gh workflow run` if triggering downstream workflows. |
| Integrity filtering | `approved` (default) for outputs that require triage+ permissions. `unapproved` or `none` when scanning community issues in the milestone — must pair with tight `safe-outputs`. See [standard guidance](../chapters/authorization-and-roles.md#standard-guidance). |

## Related events (cascade)

Deleting a milestone removes it from all assigned issues/PRs, firing `issues.demilestoned` and `pull_request.demilestoned` for every one of them in rapid succession. The `on: milestone` trigger itself does **not** fire these cascade events — but if you also have workflows listening on `issues: types: [demilestoned]` or `pull_request: types: [demilestoned]`, those fire N times in seconds when a milestone with N items is deleted. With `cancel-in-progress: true`, only the last one runs (silently dropping work for the earlier N-1); without it, runner minutes balloon. Note the naming-collision footgun: `on: milestone` is **not** `on: issues: types: [milestoned]`.

---

[← Previous: release](release.md) | [Table of Contents](../README.md) | [Next: push →](push.md)
