[← Previous: Triggers](../chapters/triggers.md) | [Table of Contents](../README.md) | [Next: `pull_request` →](pull-request.md)

# `issues`

### ✅ Recommended

> 🛑 **`issues.edited` is a privilege-escalation amplifier.** A read-only contributor can open an issue with a benign body, wait for auto-triage/auto-label workflows to bless it, then edit the issue body to inject the actual hostile payload — an `issues.edited` event fires NOW, runs the same workflow with today's secrets, today's `permissions:`. This sidesteps any human review that happened on the original `opened` event. Bonus: `issues.assigned` and `issues.labeled` events have a maintainer actor but their payload (body, title, comments) remains attacker-controlled — workflows that gate on `github.event.sender` ("only act if a maintainer triggered me") can be tricked into processing attacker prose because a maintainer applied a label. The actor changed; the data did not.

---

## Scenarios

- Auto-triage on `opened` — classify, label, assign, link related issues
- Welcome / templated reply for first-time authors
- Project-board syncing on `opened` / `closed` / `reopened`
- **Avoid** `.edited` unless you've designed for re-execution — it re-fires for every old issue edit

## Profile

| Dimension | Recommendation |
|---|---|
| `on.roles:` | Acceptable to set `all` — anyone can file an issue and workflows generally should run on all of them. Restricting to default `[admin, maintainer, write]` is a special case for workflows that should only react to maintainer-filed issues. `triage` is excluded by default — add it explicitly when triage-role users are the primary operators. |
| Activity types | `[opened]` is safest. Add `edited` only with re-execution design; never default to `[opened, edited, reopened]` without thinking. |
| Concurrency | `${{ github.workflow }}-${{ github.event.issue.number }}`. Not critical with `[opened]` only (each issue opens once). If supporting `edited`/`reopened`, determine the appropriate `cancel-in-progress` behavior — losing in-flight triage to a follow-up edit is usually worse than queuing. |
| Idempotency | **Required.** Deterministic comment markers, upsert labels, check-before-create on follow-up issues. |
| Fork posture | Apply `if: ${{ github.event_name == 'workflow_dispatch' \|\| !github.event.repository.fork }}` to prevent running within a user's fork. |
| Approval gate | Not subject to the "Approve and run workflows" button. |
| Bot/Copilot events | Issues created via `GITHUB_TOKEN` **do not** trigger `issues`. Issues via GitHub App tokens or PATs **do**. |
| Sanitize payload? | **Yes, always** in pre-agent steps. `issue.body` and `issue.title` are user-controlled; use `steps.sanitized.outputs.text`, never raw `${{ github.event.issue.body }}`. Acceptable to handle unsanitized payload within the agent job (sandboxed), coupled with proper `safe-outputs`. |
| Safe-outputs | `add-labels`, `add-comment`. Avoid `update-issue` on `.edited` (creates feedback loops). |

---

[← Previous: Triggers](../chapters/triggers.md) | [Table of Contents](../README.md) | [Next: `pull_request` →](pull-request.md)
