---
title: "discussion_comment"
---

[← Previous: `discussion`](discussion.md) | [Table of Contents](../README.md) | [Next: `labeled` / `label_command:` →](labeled-and-label-command.md)

# `discussion_comment`

### ⚠️ Use with caution

> 🛑 **The "comment on your own discussion" privilege escalation.** A read-only user can create a discussion, immediately comment on it with any payload, and edit either freely. Any agentic workflow that listens on `discussion_comment` — including `slash_command:` workflows by default — can be invoked at will by a drive-by user. The actor of the workflow run is the read-user but the actor of the workflow outputs is `github-actions[bot]` (violates [Tenet #3](../chapters/tenets.md)). If `safe-outputs:` allows `add-comment` and the comment lands on the triggering discussion, the bot output appears to endorse/reply-to the attacker prompt — lending upstream apparent authority to the attacker narrative.

---

## Scenarios

- Community Q&A responses — reply to follow-up questions on discussions
- `/command` handling in discussions via `slash_command:` (which subscribes to `discussion_comment` by default)
- Marking a discussion as answered based on comment content

**Why ⚠️:** A read-only user can create a discussion, immediately comment on it with any payload, and edit either freely. Same low-visibility concern as `discussion` — less monitored than issue/PR comments. The actor of the workflow outputs is `github-actions[bot]`, not the commenter — bot output appears to endorse/reply-to the commenter's content, lending upstream apparent authority to user-supplied narrative.

## Profile

| Dimension | Recommendation |
|---|---|
| `on.roles:` | Default `[admin, maintainer, write]`. Acceptable to set `all` for community-facing bots — same considerations as `discussion` and `issues`. |
| Activity types | `[created]` is safest. Add `edited` only if designed for re-execution — same time-bomb as `issue_comment.edited`. |
| Concurrency | `${{ github.workflow }}-${{ github.event.discussion.number }}`. Use `cancel-in-progress: false` — same reasoning as `issue_comment`: any comment on the same discussion shares the group. |
| Idempotency | **Required.** Same discipline as `issue_comment` — check before acting, claim a lock, no-op if done. gh-aw provides `lock-for-agent: true` but use with extreme caution as it prevents genuine users from interacting on the discussion while the workflow is running. |
| Fork posture | Apply `if: ${{ github.event_name == 'workflow_dispatch' \|\| !github.event.repository.fork }}` to prevent running within a user's fork. |
| Approval gate | Not subject to the "Approve and run workflows" button. |
| Bot/Copilot events | Discussion comments via `GITHUB_TOKEN` **do not** trigger. Comments via GitHub App tokens or PATs **do**. |
| Sanitize payload? | **Yes, always** in pre-agent steps. Comment body is user-controlled; use `steps.sanitized.outputs.text`, never raw `${{ github.event.comment.body }}`. Acceptable to handle unsanitized payload within the agent job (sandboxed), coupled with proper `safe-outputs`. |
| Safe-outputs | `add-comment`, `add-labels`. Audit against `on.roles:` — if `all`, every safe-output is reachable by anyone who can comment. |

---

[← Previous: `discussion`](discussion.md) | [Table of Contents](../README.md) | [Next: `labeled` / `label_command:` →](labeled-and-label-command.md)
