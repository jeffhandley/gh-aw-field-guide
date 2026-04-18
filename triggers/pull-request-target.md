<nav>

[← Previous: `pull_request`](pull-request.md) | [Table of Contents](../README.md) | [Next: `push` and `pull_request.synchronize` →](push-and-synchronize.md)

</nav>

# `pull_request_target` ⚠️

> 🛑 **`pull_request_target` is the trigger most likely to get a repo pwned, *and the approve-and-run gate is the booby trap that springs it*.** `pull_request` from forks runs sandboxed (read-only token, no secrets). Maintainers reach for `pull_request_target` to get write tokens and secrets back. The pwn vector: it checks out the **base** ref by default, but real workflows then `actions/checkout` with `ref: ${{ github.event.pull_request.head.sha }}` to see the PR's code — at which point an attacker's first PR can ship a malicious `package.json` install hook, `Makefile`, custom action under `.github/actions/`, or `.npmrc` that **executes with full repo secrets and write `GITHUB_TOKEN`**. Even without checking out PR code, the PR title/body are attacker-controlled prose the agent ingests — instant prompt injection with write tokens.
>
> **The approve-and-run gate is what makes this catastrophic, not what protects from it.** In our configuration the approval requirement is on by policy and external contributors must work from forks — so every `pull_request_target` workflow on every external PR presents the "Approve and run workflows" button, every day. That is alert fatigue by construction: the click-rate trends toward 100%, **the UI does not show what is about to execute or give guidance for what to review before approving**, and the button label conceals what is actually being authorized ("run safely-defined workflows" reads as "rubber-stamp this routine PR," not "grant this stranger's code access to every secret"). **The gate is also one-shot per click — approving *all* gated workflow runs in that batch at once, with no granular visibility or selection of which individual workflows are safe to run for this specific PR.** Click once and every gated workflow on the PR fires. **gh-aw's safe-outputs gate, output sanitization, and `staged: true` exist largely *because* of this trigger's footgun shape.**

See also: [The "Approve and run workflows" Gate](../chapters/approve-and-run-gate.md), and the [Risk Profile](../appendices/trigger-risk-profile.md) row for `pull_request_target`.

---

<nav>

[← Previous: `pull_request`](pull-request.md) | [Table of Contents](../README.md) | [Next: `push` and `pull_request.synchronize` →](push-and-synchronize.md)

</nav>
