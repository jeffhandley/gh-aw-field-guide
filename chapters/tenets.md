[← Previous: Table of Contents](../README.md) | [Table of Contents](../README.md) | [Next: Triggers →](triggers.md)

# Tenets

The design principles every workflow in this repo must satisfy. Positive framing — what we're trying to *achieve*. (The complementary negative framing — what to *avoid* — lives in the [Triggers](triggers.md) chapter and the per-trigger pages it links to.)

---

1. **We must avoid subtle privilege-escalation pitfalls — including ones a careful reviewer would miss.** This Field Guide exists because most of the pitfalls cataloged here *look* safe (gated, role-checked, fork-isolated) and only escalate under specific event combinations.

2. **Consequential actions must require human approval of a reviewable diff** with a clear audit trail and the action attributed to the human team member.

3. **Bot actions must never be presented as actions taken by a human.** The bot remains the visible *actor*; the responsible human is captured separately as the *accountable party* via co-authorship, comment metadata, or audit log.

4. **No alert fatigue, no opaque approvals.** The flip side of #2: approval must never be required for inconsequential operations, and every prompt must clearly show what will be done and why. Opaque or routine prompts cause alert fatigue and degrade #2 into theatre. (See [The "Approve and run workflows" Gate](approve-and-run-gate.md) for the canonical anti-example.)

5. **Use the right tool: deterministic by default.** Deterministic Actions and reusable workflows by default; agentic workflows only when the input is unstructured, the decision space is open, or AI unlocks a capability deterministic code cannot provide.

6. **Resist building a bespoke workflow-management platform on top of gh-aw.** Every layer of meta-orchestration becomes a product you must maintain at high opportunity cost.

7. **Limitations that exist for security reasons are not bugs to be worked around — they *are* the security model.** Bypasses (e.g., `pull_request_target` to gain write access to fork content, PAT pools used to evade bot-identity attribution, `workflow_run` to escape approval gates, `on.roles: all` to widen the actor pool) erase the protections you appear to still have. When a boundary blocks a legitimate goal, escalate it to the platform owners; do not engineer a bypass.

8. **No human single point of failure.** Operational continuity must not depend on any single team member. Credentials, approvals, and recovery knowledge must be distributed across the team — e.g., this repo's `COPILOT_PAT_*` pool distributes credential ownership and rate-limit budget across multiple team members, eliminating the burdened SPOF that a single shared PAT would create.

9. **Limit patterns; converge on canonical recipes.** Focus on canonical recipes for achieving goals consistently — especially regarding which trigger to use when. Maintain a cheat sheet that maps each scenario/goal to its trigger + conditions, and keep it current as we learn. (The [Triggers](triggers.md) chapter is the seed of that cheat sheet.)

10. **Mind the signal-to-noise ratio of high-volume triggers.** Convenience triggers (e.g., `slash_command:`) and activity-type filters (e.g., `pull_request: types: [labeled]`, `issues: types: [opened, edited]`) both compile to broad event subscriptions whose activation step runs on every matching event; the agent step fires on only a fraction of those runs. Weigh the tradeoff explicitly: **low-latency invocation requires accepting a high executed-to-filtered ratio**, and that ratio costs runner minutes (and in some configurations, model tokens) on every no-op activation. The activation step must be cheap, and the worst-case invocation rate must be estimated and acceptable.

11. **Limit the agent job to actions best suited for execution within an agent.** Keep all filtering and skipping logic in steps or jobs that execute before the agent. Execute deterministic scripts both before and after the agent job.

12. **Apply least privilege on every dimension.** Grant the bare minimum `permissions:`, `safe-outputs:`, `network.allowed:`, secrets, `tools:`, and other capabilities. Use the agent sandbox and safe-outputs controls to operate on untrusted inputs (anything authored outside the trusted maintainer set — PR bodies, comments, fork PR head SHAs, fork file contents, discussion content). **Example:** checking out a fork and executing its code is acceptable *inside* the agent sandbox; the same operation in pre- or post-agent steps executes attacker code on the runner host with full secret access and is not acceptable.

13. **Read the docs.** Even the `agentic-workflows` Copilot agent provides misleading and incorrect guidance — gh-aw releases multiple times a day, so any pattern learned from an assistant, a memory, or *this Field Guide* must be verified against the current docs at <https://gh.io/gh-aw>.

14. **Test your workflows in disposable repositories before promoting them to product repositories.** Use the [TrialOps](https://github.github.com/gh-aw/patterns/trial-ops/) pattern (`gh aw trial` creates an ephemeral private trial repo). Validate with at least one test GitHub account that has only **read** permission — that's the actor pool most likely to surface privilege-escalation and denial-of-service issues. **Never test in production.**

---

[← Previous: Table of Contents](../README.md) | [Table of Contents](../README.md) | [Next: Triggers →](triggers.md)
