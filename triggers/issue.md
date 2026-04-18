<nav>

[← Previous: Trigger Pitfalls](../chapters/trigger-pitfalls.md) | [Table of Contents](../README.md) | [Next: `pull_request` →](pull-request.md)

</nav>

# `issues`

> 🛑 **`issues.edited` is a privilege-escalation amplifier.** A read-only contributor can open an issue with a benign body, wait for auto-triage/auto-label workflows to bless it, then edit the issue body to inject the actual hostile payload — an `issues.edited` event fires NOW, runs the same workflow with today's secrets, today's `permissions:`. This sidesteps any human review that happened on the original `opened` event. Bonus: `issues.assigned` and `issues.labeled` events have a maintainer actor but their payload (body, title, comments) remains attacker-controlled — workflows that gate on `github.event.sender` ("only act if a maintainer triggered me") can be tricked into processing attacker prose because a maintainer applied a label. The actor changed; the data did not.

---

<nav>

[← Previous: Trigger Pitfalls](../chapters/trigger-pitfalls.md) | [Table of Contents](../README.md) | [Next: `pull_request` →](pull-request.md)

</nav>
