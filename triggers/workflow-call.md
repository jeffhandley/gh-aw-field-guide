<nav>

[← Previous: `milestone`](milestone.md) | [Table of Contents](../README.md) | [Next: `workflow_run` →](workflow-run.md)

</nav>

# `workflow_call`

> 🛑 **`workflow_call` is the trigger that *looks* like safe encapsulation but actually creates an undeclared trust boundary — and the called workflow runs with whatever permissions the caller had, which the callee author did not control.** A reusable agentic workflow exposes `inputs:` that downstream callers populate from `${{ github.event.* }}` — the issue body, PR title, comment text, etc. The callee author can sanitize inside the called workflow, but **they cannot enforce that callers don't pipe attacker-controlled prose straight in**. Worse for token surface area: `secrets: inherit` is the convenient knob that hands every secret in the calling repo to the called workflow — including Copilot PATs, deployment tokens, and any cross-repo secret. A maintainer adding `uses: org/shared-workflows/.github/workflows/triage.yml@main` does not see a reviewable diff of what `triage.yml` will do with their secrets; pinning to `@main` (or any moving ref) means the upstream maintainer can change the agent prompt, swap the model, or add new safe-outputs **without the calling repo seeing the change**. Pin by SHA. And note: the calling repo's branch protection / approval gates **do not apply to the called workflow's behavior** — only to the act of merging the caller's YAML.

---

<nav>

[← Previous: `milestone`](milestone.md) | [Table of Contents](../README.md) | [Next: `workflow_run` →](workflow-run.md)

</nav>
