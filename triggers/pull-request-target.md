<nav>

<a href="pull-request.md">← Previous: `pull_request`</a> | <a href="../README.md">Table of Contents</a> | <a href="push.md">Next: `push` →</a>

</nav>

# `pull_request_target` ⚠️

### ⛔ Avoid

> 🛑 **`pull_request_target` is the trigger most likely to get a repo pwned, *and the approve-and-run gate is the booby trap that springs it*.** `pull_request` from forks runs sandboxed (read-only token, no secrets). Maintainers reach for `pull_request_target` to get write tokens and secrets back. The pwn vector: it checks out the **base** ref by default, but real workflows then `actions/checkout` with `ref: ${{ github.event.pull_request.head.sha }}` to see the PR's code — at which point an attacker's first PR can ship a malicious `package.json` install hook, `Makefile`, custom action under `.github/actions/`, or `.npmrc` that **executes with full repo secrets and write `GITHUB_TOKEN`**. Even without checking out PR code, the PR title/body are attacker-controlled prose the agent ingests — instant prompt injection with write tokens.
>
> **The approve-and-run gate is what makes this catastrophic, not what protects from it.** In our configuration the approval requirement is on by policy and external contributors must work from forks — so every `pull_request_target` workflow on every external PR presents the "Approve and run workflows" button, every day. That is alert fatigue by construction: the click-rate trends toward 100%, **the UI does not show what is about to execute or give guidance for what to review before approving**, and the button label conceals what is actually being authorized ("run safely-defined workflows" reads as "rubber-stamp this routine PR," not "grant this stranger's code access to every secret"). **The gate is also one-shot per click — approving *all* gated workflow runs in that batch at once, with no granular visibility or selection of which individual workflows are safe to run for this specific PR.** Click once and every gated workflow on the PR fires. **gh-aw's safe-outputs gate, output sanitization, and `staged: true` exist largely *because* of this trigger's footgun shape.**

See also: [The "Approve and run workflows" Gate](../chapters/approve-and-run-gate.md), and the [Risk Profile](../appendices/trigger-risk-profile.md) row for `pull_request_target`.

---

## Scenarios

- Metadata-only operations on fork PRs that require a write token — labeling based on path globs, posting a comment with CI results, dependency-review on the diff metadata.
- Reprocessing when PR head changes (via `synchronize`) — e.g., clearing "author action needed" state when the author pushes a fix.
- **Never** for anything that checks out or executes PR head code.

**Why ⛔:** Runs on the **base ref** with **full `GITHUB_TOKEN` and access to all secrets**, even for fork PRs. This is the trigger most likely to get a repo pwned. The approval gate compounds the danger — clicking approves *all* gated workflows, including this one with full secrets. Assume the button will always be clicked — by someone trying to approve a different workflow. Same as `pull_request`, the button is presented even if the contributor doesn't match `on.roles:` and the workflow will immediately exit.

**Recommended alternatives:**
- **[`schedule`](schedule.md)** — preferred. Not subject to event spamming or user-triggered invocations. Acts as a polling job. Subject to the same idempotency requirements as other alternatives, but concurrency is substantially more straightforward.
- **[`issue_comment` / `slash_command:`](comment-and-slash-command.md)** — for on-demand PR operations triggered by a maintainer's `/command`. Not subject to the approval gate.
- **[`pull_request: types: [labeled]` / `label_command:`](labeled-and-label-command.md)** — for PR operations triggered by a triage+ user applying a label.

## Profile

| Dimension | Recommendation |
|---|---|
| `on.roles:` | Default `[admin, maintainer, write]`. *Can* use `triage` or `all`, but with ☢️ extreme caution — only if the workflow is safe taking the same action for a read-only user's PR as it would for a maintainer-submitted PR. |
| Activity types | Narrowest possible. `[opened]` or `[labeled]` for metadata operations. `synchronize` is valid when you need to reprocess on head-change (e.g., clearing "author action needed" state), but understand it fires on every push. **Avoid** `edited` (same re-execution risk as `issues.edited`, but now with secrets). |
| Concurrency | `${{ github.workflow }}-${{ github.event.pull_request.number }}`. `cancel-in-progress` depends on idempotency considerations and whether output from previous runs would be discarded/overwritten. |
| Idempotency | **Required.** Every mutation must be safe to repeat — partial-write from a canceled run + retry must converge to the same end state. Note that several `synchronize` events commonly fire in quick succession (force-push, rebase, rapid commits), which is another reason why a schedule or other trigger is preferred over immediate invocation. |
| Fork posture | Apply `if: ${{ github.event_name == 'workflow_dispatch' \|\| !github.event.repository.fork }}` to prevent running within a user's fork. Additionally, gate the agent step on `github.event.pull_request.head.repo.fork == false` if the workflow should refuse cross-fork PRs entirely. **Never check out the PR head SHA** in any job that has secrets in its environment. |
| Approval gate | **Subject** — and this is where the gate is most dangerous. Clicking approves *all* gated workflows, including this one with full secrets. Assume the button will always be clicked — by someone trying to approve a different workflow. |
| Bot/Copilot events | Events created via `GITHUB_TOKEN` **do not** trigger `pull_request_target`. Events from GitHub App tokens or PATs **do**. |
| Sanitize payload? | **Yes, critically** in pre-agent steps. PR title, body, branch name, label names, and commit messages are all attacker-controlled; use `steps.sanitized.outputs.text`, never raw `${{ github.event.pull_request.body }}`. However, it *is* acceptable to handle the unsanitized payload within the **agent job** itself, as that job is sandboxed — this must be coupled with proper `safe-outputs` handling so the agent cannot exfiltrate or write beyond its declared outputs. |
| Safe-outputs | `add-labels`, `add-comment` only. **Avoid** `push-to-pull-request-branch`, `create-pull-request`, `update-issue` — each is a write amplifier with full secrets. |

---

<nav>

<a href="pull-request.md">← Previous: `pull_request`</a> | <a href="../README.md">Table of Contents</a> | <a href="push.md">Next: `push` →</a>

</nav>
