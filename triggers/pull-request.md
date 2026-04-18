<nav>

[← Previous: `issues`](issue.md) | [Table of Contents](../README.md) | [Next: `pull_request_target` →](pull-request-target.md)

</nav>

# `pull_request`

Two distinct headline footguns live here. (The third — per-push stacking on `synchronize` — has its own page in [`push` and `pull_request.synchronize`](push-and-synchronize.md).)

> 🛑 **A. `closed` ≠ `merged`.** Workflows that celebrate a merge, post release notes, file follow-up issues, or kick off downstream automation on `pull_request.closed` fire on PR-closed-without-merge too, unless they explicitly check `github.event.pull_request.merged == true`. The PR author can game this: open a PR, close it, and the "thanks for merging!" workflow runs.

> 🛑 **B. The `reopened` time-bomb.** Anyone with write access (and the original PR author themselves) can reopen a closed PR from 2 years ago. With default `types: [opened, synchronize, reopened]`, the workflow fires today, with today's secrets and today's `permissions:`, against the 2-year-old PR head SHA. The activation step has no concept of "this PR content was reviewed in 2023; the world has changed since." Combine with `pull_request_target` and this is catastrophic.

---

<nav>

[← Previous: `issues`](issue.md) | [Table of Contents](../README.md) | [Next: `pull_request_target` →](pull-request-target.md)

</nav>
