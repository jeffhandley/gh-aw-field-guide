<nav>

[← Previous: `discussion`](discussion.md) | [Table of Contents](../README.md) | [Next: `pull_request_review_comment` →](pull-request-review-comment.md)

</nav>

# `discussion_comment`

> 🛑 **The "comment on your own discussion" privilege escalation.** A read-only user can create a discussion, immediately comment on it with any payload, and edit either freely. Any agentic workflow that listens on `discussion_comment` — including `slash_command:` workflows by default — can be invoked at will by a drive-by user. The actor of the workflow run is the read-user but the actor of the workflow outputs is `github-actions[bot]` (violates [Tenet #3](../chapters/tenets.md)). If `safe-outputs:` allows `add-comment` and the comment lands on the triggering discussion, the bot output appears to endorse/reply-to the attacker prompt — lending upstream apparent authority to the attacker narrative.

---

<nav>

[← Previous: `discussion`](discussion.md) | [Table of Contents](../README.md) | [Next: `pull_request_review_comment` →](pull-request-review-comment.md)

</nav>
