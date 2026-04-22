---
title: "pull_request"
---

[← Previous: workflow_run](workflow-run.md) | [Table of Contents](../README.md) | [Next: pull_request_target →](pull-request-target.md)

# `pull_request`

### ⛔ Avoid

> 🛑 **A. `closed` ≠ `merged`.** Workflows that celebrate a merge, post release notes, file follow-up issues, or kick off downstream automation on `pull_request.closed` fire on PR-closed-without-merge too, unless they explicitly check `github.event.pull_request.merged == true`. The PR author can game this: open a PR, close it, and the "thanks for merging!" workflow runs.

> 🛑 **B. The `reopened` time-bomb.** Anyone with write access (and the original PR author themselves) can reopen a closed PR from 2 years ago. With default `types: [opened, synchronize, reopened]`, the workflow fires today, with today's secrets and today's `permissions:`, against the 2-year-old PR head SHA. The activation step has no concept of "this PR content was reviewed in 2023; the world has changed since." Combine with `pull_request_target` and this is catastrophic.

---

## Scenarios

- CI / build / test on every PR (the canonical CI trigger)
- Deterministic labeling based on changed paths
- Code-review assist (agentic review comments on the diff)
- **The safer of the two PR triggers** — read-only `GITHUB_TOKEN`, no secrets for fork PRs. If you must react to PR events directly, this is the one to use over `pull_request_target`.

**Why ⛔:** The existence of *any* `pull_request` workflow causes the "Approve and run workflows" button to appear for outside contributors' PRs (including Copilot), and clicking the button approves *all* gated workflows — including any `pull_request_target` workflows in the repo. The trigger can be safely used, even with fork PRs (the token is already read-only, there are no secrets), but the button's existence creates alert fatigue that undermines the overall security model. **The button is presented even if the user does not match `on.roles:` and the workflow will immediately exit.**

**Recommended alternatives:**
- **[`schedule`](schedule.md)** — for periodic PR housekeeping (stale PR sweeps, label audits) that doesn't need to react in real-time.
- **[`pull_request: types: [labeled]` / `label_command:`](labeled-and-label-command.md)** — for PR operations triggered by a triage+ user applying a label.

## Profile

| Dimension | Recommendation |
|---|---|
| `on.roles:` | Acceptable to set `all` — anyone can create a pull request and workflows generally should run on all of them. Restricting to default `[admin, maintainer, write]` is a special case. `triage` is excluded by default — add explicitly when triage users open PRs from branches. |
| Activity types | Default `[opened, synchronize, reopened]` is correct for CI. `synchronize` fires once per push to the PR branch (including force-pushes) — valid for reprocessing when the head changes but note that several events commonly fire in quick succession. Add `ready_for_review` if you skip draft PRs. **Avoid** `edited` unless you need to re-react to title/body changes. |
| Concurrency | `${{ github.workflow }}-${{ github.event.pull_request.number }}` with `cancel-in-progress: true` — the standard CI pattern where only the latest push matters. For agentic workflows that post comments or apply labels, `cancel-in-progress: false` is safer to avoid partial-write races. |
| Idempotency | **Recommended.** Upsert comments (find-and-update rather than post-new), upsert labels, upsert status checks. |
| Fork posture | Apply `if: ${{ github.event_name == 'workflow_dispatch' \|\| !github.event.repository.fork }}` to prevent running within a user's fork. Fork PRs *to upstream* are safe (read-only token, no secrets). |
| Approval gate | **Subject** — outside contributors' PRs trigger the "Approve and run workflows" button, **even when the contributor doesn't match `on.roles:` and the workflow will immediately exit.** The existence of *any* `pull_request` workflow causes this button to appear, and clicking approves *all* gated workflows. |
| Bot/Copilot events | Events created via `GITHUB_TOKEN` **do not** trigger `pull_request`. Events from GitHub App tokens or PATs **do**. |
| Sanitize payload? | **Yes, always** in pre-agent steps. PR title, body, and branch name are user-controlled; use `steps.sanitized.outputs.text`, never raw `${{ github.event.pull_request.body }}`. Acceptable to handle unsanitized payload within the agent job (sandboxed), coupled with proper `safe-outputs`. |
| Safe-outputs | `add-labels`, `add-comment` for review-assist. `create-issue` if filing follow-ups. No `push-to-pull-request-branch` unless the workflow explicitly intends to commit changes. |
| Integrity filtering | `approved`. Fork contributors are typically `CONTRIBUTOR` or lower, so their PR content is filtered out before the agent sees it. See [standard guidance](../chapters/authorization-and-roles.md#standard-guidance). |

---

[← Previous: workflow_run](workflow-run.md) | [Table of Contents](../README.md) | [Next: pull_request_target →](pull-request-target.md)
