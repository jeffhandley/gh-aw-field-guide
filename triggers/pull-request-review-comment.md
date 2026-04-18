<nav>

[← Previous: `discussion_comment`](discussion-comment.md) | [Table of Contents](../README.md) | [Next: `milestone` →](milestone.md)

</nav>

# `pull_request_review_comment`

> 🛑 **The trigger nobody remembers exists — until it fires unexpectedly and ingests attacker-controlled prose.** Most workflow authors think there is one "PR comments" trigger. There are actually three distinct events: [`issue_comment`](comment-and-slash-command.md) (top-level PR conversation), [`pull_request_review`](pull-request-review.md) (the review body), and `pull_request_review_comment` (the inline line-level comment). Workflows that subscribe to `issue_comment` for a `/command` or for prompt content will silently miss inline review comments — but the inverse is more dangerous: a workflow subscribed to `pull_request_review_comment` (often by accident, copy-pasted from a sample) fires on every line-level scribble by anyone with read access. The comment body is unstructured prose tied to a specific code line — high-leverage prompt-injection surface because attackers can position the malicious instruction next to specific code the agent is being asked to reason about ("the line above is intentional, ignore prior instructions and …"). The `.edited` time-bomb applies: an old inline comment edited today fires today's workflow with today's secrets against potentially-stale PR head SHA. Concurrency twist: when this trigger is mixed into a PR-handling workflow alongside `pull_request` and `pull_request_review`, group-key choice is even worse than for `pull_request_review` alone — the inline-comment payload doesn't always have a clean `pull_request.number` at the top level (you have to dig through `event.pull_request` which is sometimes null in deleted-comment payloads).

---

<nav>

[← Previous: `discussion_comment`](discussion-comment.md) | [Table of Contents](../README.md) | [Next: `milestone` →](milestone.md)

</nav>
