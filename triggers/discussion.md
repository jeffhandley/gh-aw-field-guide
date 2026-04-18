<nav>

[← Previous: `workflow_dispatch`](workflow-dispatch.md) | [Table of Contents](../README.md) | [Next: `discussion_comment` →](discussion-comment.md)

</nav>

# `discussion`

> 🛑 **`discussion` is the most-open untrusted-input surface in the GitHub model and it has no approval gate.** Anyone with repository read access can fire `discussion.created` or `discussion.edited`. The workflow runs with its declared `permissions:` and `safe-outputs:`, which the author thinks of as "what we can do" but is actually "what any read-user can cause us to do." Discussion bodies are unstructured prose — an obvious prompt-injection vector. The default `on.roles:` allowlist stops the agent step for read-role actors, but the workflow still queues a runner ([Tenet #10](../chapters/tenets.md)), and any pre-agent step that does something with the discussion content runs anyway.

---

<nav>

[← Previous: `workflow_dispatch`](workflow-dispatch.md) | [Table of Contents](../README.md) | [Next: `discussion_comment` →](discussion-comment.md)

</nav>
