<nav>

[← Previous: `schedule`](schedule.md) | [Table of Contents](../README.md) | [Next: `discussion` →](discussion.md)

</nav>

# `workflow_dispatch`

> 🛑 **`workflow_dispatch` is the most-trusted trigger we have — and that trust gets quietly extended to anyone with write access, who can run *any* dispatchable workflow against *any* branch with *any* `inputs:` payload.** A maintainer adds `workflow_dispatch:` because they want a manual escape hatch for themselves. Every other write-role contributor inherits that same button. With `inputs:` defined as free-form strings, write-role users can supply attacker-style prose to an agent without leaving any of the usual artifacts (no PR, no comment, no commit) — the entire payload lives in the workflow run's "Inputs" section that nobody scrolls back to read. **Branch selection is also user-controlled**: a write-role contributor can run the dispatch against a branch they just pushed (or a long-stale branch) where the workflow YAML is **different** from `main`'s — meaning the version of the workflow they're invoking may have less-restrictive `permissions:`, weaker `safe-outputs.allowed:`, or an attacker-friendlier prompt. The `main`-branch version is not what runs; the **selected branch's** version is.

---

<nav>

[← Previous: `schedule`](schedule.md) | [Table of Contents](../README.md) | [Next: `discussion` →](discussion.md)

</nav>
