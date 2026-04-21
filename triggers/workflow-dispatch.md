---
title: "workflow_dispatch"
---

[← Previous: schedule](schedule.md) | [Table of Contents](../README.md) | [Next: release →](release.md)

# `workflow_dispatch`

### ✅ Recommended

> 🛑 **`workflow_dispatch` is the most-trusted trigger we have — and that trust gets quietly extended to anyone with write access, who can run *any* dispatchable workflow against *any* branch with *any* `inputs:` payload.** A maintainer adds `workflow_dispatch:` because they want a manual escape hatch for themselves. Every other write-role contributor inherits that same button. With `inputs:` defined as free-form strings, write-role users can supply attacker-style prose to an agent without leaving any of the usual artifacts (no PR, no comment, no commit) — the entire payload lives in the workflow run's "Inputs" section that nobody scrolls back to read. **Branch selection is also user-controlled**: a write-role contributor can run the dispatch against a branch they just pushed (or a long-stale branch) where the workflow YAML is **different** from `main`'s — meaning the version of the workflow they're invoking may have less-restrictive `permissions:`, weaker `safe-outputs.allowed:`, or an attacker-friendlier prompt. The `main`-branch version is not what runs; the **selected branch's** version is.

---

## Scenarios

- Manual escape hatch for any workflow — debugging, catch-up runs, ad-hoc invocation
- **Automatically paired with most gh-aw triggers** — `slash_command:`, `label_command:`, schedule shorthands, and trigger shorthands all auto-inject `workflow_dispatch` by default
- Administrative operations that should only be invoked deliberately (e.g., release cutting, environment provisioning)
- The explicit escape hatch for the fork guard pattern — fork owners can manually dispatch to intentionally test workflows in their fork
- Passing free-form prompt content into agentic workflows via string `inputs:` — the invoker is a write+ user whose input is trusted

## Profile

| Dimension | Recommendation |
|---|---|
| `on.roles:` | N/A — only write+ users can invoke `workflow_dispatch`. gh-aw does not inject a role check for `workflow_dispatch`-only workflows. |
| Activity types | N/A. Define `inputs:` with constrained types (choice, boolean, number) for structured parameters, and free-form strings for prompt content to pass into agentic workflows. **Strongly recommended to pair with other triggers** (schedule shorthands, `slash_command:`, etc.) — gh-aw auto-injects `workflow_dispatch` for most trigger types. |
| Concurrency | `${{ github.workflow }}-${{ github.run_id }}` for one-off invocations. For workflows that also have event-driven triggers, match the concurrency group of the primary trigger. |
| Idempotency | **Recommended.** Manual invocations may be retried or fired multiple times by different users. |
| Fork posture | Apply `if: ${{ github.event_name == 'workflow_dispatch' \|\| !github.event.repository.fork }}` — fork owners can dispatch in their own fork against any branch. This is intentional — `workflow_dispatch` is the explicit escape hatch in the fork guard pattern. |
| Approval gate | Not subject to the "Approve and run workflows" button. |
| Bot/Copilot events | Dispatches via `GITHUB_TOKEN` **do not** trigger. Dispatches via GitHub App tokens or PATs **do** — this is the canonical way for one workflow to invoke another. |
| Sanitize payload? | **Yes** for free-form string `inputs:` in pre-agent steps. Use `steps.sanitized.outputs.text` or pass inputs only via `env:` blocks, never directly into shell commands. Acceptable to handle unsanitized inputs within the agent job (sandboxed), coupled with proper `safe-outputs`. **Branch selection is also user-controlled** — a write-role contributor can dispatch against a branch with a different version of the workflow YAML that has less-restrictive permissions or safe-outputs. |
| Safe-outputs | Depends on workflow purpose. No inherent restriction — but audit the blast radius given that any write-role contributor can invoke with any inputs against any branch. |
| Integrity filtering | `approved` (default) for outputs that require triage+ permissions. `unapproved` or `none` when the dispatched workflow intentionally consumes community content — must pair with tight `safe-outputs`. See [standard guidance](../chapters/authorization-and-roles.md#standard-guidance). |

---

[← Previous: schedule](schedule.md) | [Table of Contents](../README.md) | [Next: release →](release.md)
