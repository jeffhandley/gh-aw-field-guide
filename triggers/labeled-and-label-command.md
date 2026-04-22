---
title: "labeled and label_command"
---

[← Previous: schedule](schedule.md) | [Table of Contents](../README.md) | [Next: issues →](issue.md)

# `pull_request: types: [labeled]` and `label_command:`

### ✅ Recommended

> 🛑 **`label_command:` turns labels into one-shot RPC calls — and inherits the same broad-subscription concurrency catastrophe as `slash_command:`.** The trigger fires when any write-role contributor applies the configured label, runs the agentic workflow, and then **automatically removes the label**. The audit trail is preserved but is *subtle* — the issue/PR event stream shows `Octocat added the deploy label` immediately followed by `github-actions[bot] removed the deploy label`, which is easy to scroll past as a label-fix correction. For a `/deploy`-shaped command, this means a triage-role contributor (write access but typically not deploy-trusted) can fire the deploy by applying a label, and the only record of the command being issued lives in two adjacent timeline events that look like noise.
>
> The bigger structural problem is the same shape as [`slash_command:`](comment-and-slash-command.md): `label_command:` subscribes to *every* `labeled` event on the configured event types (`issues`, `pull_request`, `discussion`). The workflow runs — consuming a runner — on every label application by every write-role user, and only the name match causes early abort. Like `slash_command:`, this means concurrency groups cannot safely be used with `cancel-in-progress: true` (a benign label change on the same target would cancel the in-flight command). Therefore `label_command:` workflows **MUST be implemented to be idempotent**: check before acting, claim a lock, no-op if the work is already in progress or already done.

> 🛑 **The `labeled` activity type fires under the labeller's identity, but the payload remains attacker-controlled.** A workflow gated on `github.event.sender` ("only act if a maintainer triggered me") can be tricked: a read-only user opens an issue with attacker prose, a triage-role maintainer applies the `bug` label (innocuous), and the agent now runs as if the maintainer authored the issue. The actor changed; the body, title, and comments did not.

---

## Scenarios

- One-shot agentic operations triggered by applying a label — `/backport`, `/approve-release`, `/run-benchmarks`
- State-driven workflows where a label represents a stage gate (e.g., `ready-for-review`, `needs-backport`)
- **Higher authorization floor than `slash_command:`** — applying a label requires triage+ role, which is a meaningful step up from read-only comment access
- `label_command:` with `remove_label: false` turns the label into a persistent state marker rather than a one-shot command

## Profile

| Dimension | Recommendation |
|---|---|
| `on.roles:` | Default `[admin, maintainer, write]`. **`triage` is the natural fit here** — label application requires triage role, so excluding triage from `on.roles:` means the activation job *fires* (consuming a runner) but the agent step is *denied*. Add `triage` explicitly for most label-driven workflows. |
| Activity types | For raw `issues`/`pull_request`/`discussion`: `types: [labeled]` only. For `label_command:`: configured via the label name; `events:` defaults to `[issues, pull_request, discussion]` — narrow if only needed on one. |
| Concurrency | `${{ github.workflow }}-${{ github.event.issue.number }}`. Use `cancel-in-progress: false` — same shape as `slash_command:`: any label application on the same target shares the group, so a benign label change would cancel an in-flight command. |
| Idempotency | **Required.** Same discipline as `slash_command:` — check before acting, claim a lock, no-op if already done. Bulk label operations (via GitHub UI or scripts) fire one event per item in rapid succession. `label_command:` auto-removes the label after activation, providing a natural one-shot signal, but the workflow must still be safe to re-run if the label is re-applied. |
| Fork posture | Apply `if: ${{ github.event_name == 'workflow_dispatch' \|\| !github.event.repository.fork }}` to prevent running within a user's fork. Labels are upstream-only, but forked repos have their own label sets where these workflows would fire. |
| Approval gate | Not subject to the "Approve and run workflows" button. |
| Bot/Copilot events | Label applications via `GITHUB_TOKEN` **do not** trigger `labeled`. Label applications via GitHub App tokens or PATs **do**. |
| Sanitize payload? | **Yes** in pre-agent steps. The label name is constrained, but the issue/PR body, title, and comments remain attacker-controlled; use `steps.sanitized.outputs.text`, never raw `${{ github.event.issue.body }}`. Acceptable to handle unsanitized payload within the agent job (sandboxed), coupled with proper `safe-outputs`. |
| Safe-outputs | Depends on command purpose. `add-labels`, `add-comment` for state transitions. Audit against `on.roles:` — triage users can apply labels, so every safe-output is reachable by the triage role if included. |
| Integrity filtering | `none` — the label application by a triage+ user *is* the human-in-the-loop gate, replacing the need for integrity filtering to gate content trust. Must pair with tight `safe-outputs` and `on.roles:` including `triage`. See [standard guidance](../chapters/authorization-and-roles.md#standard-guidance). |

---

[← Previous: schedule](schedule.md) | [Table of Contents](../README.md) | [Next: issues →](issue.md)
