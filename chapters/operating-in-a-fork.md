<nav>

<a href="approve-and-run-gate.md">← Previous: The "Approve and run workflows" Gate</a> | <a href="../README.md">Table of Contents</a> | <a href="concurrency-and-races.md">Next: Concurrency and Race Conditions →</a>

</nav>

# Operating Within a Fork ⚠️

When you fork a repository, a copy of every workflow file (`.github/workflows/**`) comes with the fork — including agentic workflows. **GitHub then treats your fork as its own first-class repository.** Any event you trigger *within your fork* fires the workflow inside *your fork*, with *your fork's* secrets and *your fork's* `GITHUB_TOKEN`. This is structurally separate from the cross-fork PR scenario in [The "Approve and run workflows" Gate](approve-and-run-gate.md) and is frequently a surprise.

## What fires when you operate inside your own fork

| Activity inside your fork | Workflow fires? | Runs as | Token / secrets |
|---|---|---|---|
| You push commits to a branch in your fork | ✅ on `push` | You (fork owner) | Your fork's secrets, full write `GITHUB_TOKEN` to your fork |
| You open a PR `feature → main` **within your fork** (self-review pattern) | ✅ on `pull_request` (this is **not** treated as a cross-fork PR — both refs are in your fork) | You | Your fork's secrets, full write token |
| You open an issue, comment, react in your fork | ✅ on `issues`, `issue_comment`, etc. | You | Your fork's secrets, full write token |
| You apply a slash command in your own fork | ✅ — and **`on.roles:` defaults still apply**, but you're admin of your own fork, so the membership check passes | You | Your fork's secrets, full write token |
| Scheduled (cron) workflows in your fork | ❌ by default — GitHub disables `schedule` triggers on forks until you re-enable them in the Actions tab | n/a | n/a |
| You open a PR from your fork **back to upstream** | ✅ on upstream's `pull_request` (with read-only token, no upstream secrets); ✅ on upstream's `pull_request_target` 🛑 (with full upstream secrets) | Upstream | Upstream's secrets (with the cross-fork caveats from [`pull_request`](../triggers/pull-request.md) / [`pull_request_target`](../triggers/pull-request-target.md)) |

**Key consequence:** Workflows you forked from upstream — agentic or otherwise — will **start running for you on routine activity in your fork**, often unexpectedly. They run as *you* (or whatever PAT pool *you* configured), not as the upstream owner. This is harmless when secrets are unset (most workflows fail fast), but is an unwanted surprise when:

- The forked workflow contains a `bash` step that pushes to a remote, posts to Slack, files a GitHub issue, or sends an email.
- The forked workflow uses GitHub Apps that you happened to install on your fork.
- You're using your fork as a sandbox to "study" the upstream workflows and they fire while you read.
- You reuse the upstream's PAT-pool secret names and have analogous secrets in your fork — your secrets get used by code you didn't author.

## The `if: workflow_dispatch || not-a-fork` guard pattern

The simplest and most reliable defense is a top-level job condition that prevents every workflow from running in any fork unless explicitly invoked manually:

```yaml
on:
  pull_request:
  push:
  issue_comment:
  workflow_dispatch:

jobs:
  guard:
    if: ${{ github.event_name == 'workflow_dispatch' || github.event.repository.fork == false }}
    # ...rest of workflow
```

Equivalent and arguably stricter — pin the workflow to the **specific upstream `owner/repo`**:

```yaml
if: ${{ github.event_name == 'workflow_dispatch' || github.repository == 'OWNER/REPO' }}
```

**Why both clauses?**

- `github.event.repository.fork == false` (or `github.repository == 'OWNER/REPO'`) prevents the workflow from running on routine events inside any fork.
- `github.event_name == 'workflow_dispatch'` is the explicit escape hatch: if a forker *intends* to run the workflow themselves to test their changes, they invoke it manually from the Actions tab. They get the consequences of their own action, not a surprise.

**For agentic workflows specifically:** add the same `if:` to the `pre-activation` job in the auto-injected `on.steps:` block, so even the activation runner doesn't spin up in forks. Without this, the workflow still consumes a runner minute in every fork on every event — and if the fork owner happens to have analogously-named secrets, the agent step itself may run.

> 💡 **Idiomatic placement.** Apply the guard at the *job* level (or on the activation/pre-activation job for gh-aw), **not** the step level. A job-level `if:` skips the entire job (no runner spun up); a step-level `if:` still consumes a runner.

## What the fork owner *cannot* do to you (the upstream)

Forking does not give the fork owner any new privileges over the upstream. They still have to come back through the cross-fork PR gate ([The "Approve and run workflows" Gate](approve-and-run-gate.md)) to affect upstream. The risks here are entirely **about the fork owner being surprised by their own forked copies of your workflows**, and (transitively) about the *upstream* being judged poorly for shipping workflows that misbehave in forks.

---

<nav>

<a href="approve-and-run-gate.md">← Previous: The "Approve and run workflows" Gate</a> | <a href="../README.md">Table of Contents</a> | <a href="concurrency-and-races.md">Next: Concurrency and Race Conditions →</a>

</nav>
