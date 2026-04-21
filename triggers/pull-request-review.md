---
title: "pull_request_review"
---

[← Previous: pull_request: types: [labeled]` and `label_command:`](labeled-and-label-command.md) | [Table of Contents](../README.md) | [Next: pull_request_review_comment →](pull-request-review-comment.md)

# `pull_request_review`

### ⚠️ Use with caution

> 🛑 **"A review was submitted" is not "the PR was approved" — but workflows conflate the two.** The event fires for any review type, including `COMMENT`-type reviews that any read-role user (or any drive-by GitHub account) can submit. Workflows that auto-merge "after N reviews submitted," post "ready to land" comments, or kick off deployment on review-submitted fire for non-approving comments too — including comments authored by accounts with no write access at all. Even gating correctly on `event.review.state == 'approved'` doesn't suffice because actor authorization is independent of review state. Review bodies are unstructured prose authored by anyone with read access — an under-monitored prompt-injection vector that workflows ingest as if it came from a trusted reviewer. `pull_request_review.edited` fires today against today's secrets when an old review is edited (same time-bomb shape as `issue_comment.edited`).
>
> *Concurrency twist:* `pull_request_review` is rarely the only trigger on a PR-handling workflow; it usually shares a workflow file with `pull_request` and `issue_comment`. If they share `concurrency.group: pr-${{ github.event.pull_request.number || github.event.issue.number }}` with `cancel-in-progress: true`, a stray comment or a reviewer dismissing an old review can cancel a half-finished APPROVE-handling run — the activation passed, the agent is doing real work, then a benign comment kills it mid-flight, leaving the PR in an indeterminate state (label half-applied, status check half-posted). Without `cancel-in-progress`, two near-simultaneous reviewers stack runs and post duplicate comments or apply duplicate labels. There is no group key that gets this right for all three triggers in one workflow.

---

## Scenarios

- Auto-merge after receiving the required number of approvals (gate on `event.review.state == 'approved'` *and* verify the reviewer has write+ role)
- Post-approval automation — apply "ready to land" label, kick off pre-merge checks, notify downstream teams
- Review-driven triage — route PRs to specific teams based on who reviewed

**Why ⚠️:** "A review was submitted" is not "the PR was approved." The event fires for any review type including `COMMENT`-type reviews that any read-role user can submit. This trigger is rarely standalone; it usually shares a workflow with `pull_request` and `issue_comment`, creating a multi-trigger concurrency nightmare.

## Profile

| Dimension | Recommendation |
|---|---|
| `on.roles:` | Default `[admin, maintainer, write]`. `triage` excluded by default — add only if triage-role reviews should trigger automation. Acceptable to set `all` if the workflow is safe processing reviews from any contributor — same considerations as `issues` or `issue_comment`. |
| Activity types | `[submitted]` for approval-gated workflows. Add `dismissed` if you need to react to approval revocation. **Avoid** `edited` — same time-bomb as `issue_comment.edited` (editing an old review fires today with today's secrets). |
| Concurrency | `${{ github.workflow }}-${{ github.event.pull_request.number }}`. Use `cancel-in-progress: false` — if this trigger shares a workflow with `pull_request` or `issue_comment`, a stray comment or review dismissal can cancel a half-finished approval-handling run. There is no group key that gets this right for all three triggers in one workflow — prefer separate workflows per trigger. |
| Idempotency | **Required.** Two near-simultaneous reviewers stack runs that post duplicate comments or apply duplicate labels. |
| Fork posture | Apply `if: ${{ github.event_name == 'workflow_dispatch' \|\| !github.event.repository.fork }}` to prevent running within a user's fork. |
| Approval gate | Not directly subject to the "Approve and run workflows" button (that's on `pull_request`/`pull_request_target`). However, if the workflow *also* subscribes to `pull_request`, the gate applies to those events. |
| Bot/Copilot events | Reviews via `GITHUB_TOKEN` **do not** trigger. Reviews via GitHub App tokens or PATs **do**. |
| Sanitize payload? | **Yes, always** in pre-agent steps. Review body is user-controlled; use `steps.sanitized.outputs.text`, never raw `${{ github.event.review.body }}`. Acceptable to handle unsanitized payload within the agent job (sandboxed), coupled with proper `safe-outputs`. |
| Safe-outputs | `add-labels`, `add-comment` for post-approval workflows. Avoid `push-to-pull-request-branch` or `create-pull-request` — over-broad for review-driven automation. |
| Integrity filtering | `approved`. `unapproved` or `none` when intentionally consuming community review content — must pair with tight `safe-outputs`. See [standard guidance](../chapters/authorization-and-roles.md#standard-guidance). |

---

[← Previous: pull_request: types: [labeled]` and `label_command:`](labeled-and-label-command.md) | [Table of Contents](../README.md) | [Next: pull_request_review_comment →](pull-request-review-comment.md)
