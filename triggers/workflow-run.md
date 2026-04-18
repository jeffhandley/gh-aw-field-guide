<nav>

[← Previous: `workflow_call`](workflow-call.md) | [Table of Contents](../README.md) | [Next: The Two Categories of Triggers →](../chapters/trigger-categories.md)

</nav>

# `workflow_run` ⚠️

> 🛑 **`workflow_run` is [`pull_request_target`](pull-request-target.md)'s quieter sibling: it launders untrusted output from a fork PR's sandboxed build into a privileged context with full secrets — and nobody reviewing the upstream workflow knows that's about to happen.** The pattern that gets repos pwned: a `pull_request` workflow runs the fork's tests in a sandbox (correct), uploads test results / coverage reports / built artifacts as workflow artifacts (still fine), and then a separate `workflow_run`-triggered workflow downloads those artifacts and acts on them — posting a comment, updating a status check, deploying a preview, or feeding the artifact contents to an agent. **The artifact came from attacker-controlled fork code; the consuming workflow has full write tokens and secrets.** A malicious PR uploads an artifact named `coverage.json` whose contents include shell-injection or prompt-injection payloads and the consuming workflow shells out unsafely, or feeds the JSON to an agent that ingests the prompt-injection. There is no UI signal connecting the upstream PR to the downstream `workflow_run`; reviewers see "tests passed on a fork PR" and miss that a privileged workflow then ran with that PR's data. And `workflow_run` does **not** inherit the upstream's `pull_request` head SHA — the consuming workflow runs against the **default branch's** YAML, so attackers can rely on the production version of the consumer workflow even when the producer workflow on their fork is sandboxed.

---

<nav>

[← Previous: `workflow_call`](workflow-call.md) | [Table of Contents](../README.md) | [Next: The Two Categories of Triggers →](../chapters/trigger-categories.md)

</nav>
