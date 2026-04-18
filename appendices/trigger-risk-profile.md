<nav>

[← Previous: Authorization, Roles, and Read-Only Contributors](../chapters/authorization-and-roles.md) | [Table of Contents](../README.md) | [Next: Appendix B — Footnotes →](footnotes.md)

</nav>

# Appendix A: Trigger-by-Trigger Risk Profile

The dimensions to compare across triggers: **security/authz**, **approve-and-run gate**, **when it runs (runner & token cost)**, **concurrency/race**, **unintended consequences**, **forks**, **mixing with `workflow_dispatch`**.

This appendix covers **every** trigger including the ones with no standalone pitfall page. Triggers with their own page are linked.

## [`workflow_dispatch`](../triggers/workflow-dispatch.md)
- **Authz:** Write access required. Cannot be invoked by readers/forks.
- **Approve gate:** N/A.
- **Runs:** Only when explicitly invoked or when paired with another trigger that auto-injects it (most gh-aw shorthands do).
- **Concurrency:** Standard. Manually-fired runs respect groups.
- **Forks:** Cannot dispatch on a fork's branch from upstream; the fork owner can dispatch in their own fork.
- **Unintended:** Inputs are user-controlled strings → never pass them un-quoted into shell. Use `${{ github.event.inputs.x }}` only inside `env:` blocks.

## [`schedule`](../triggers/schedule.md)
- **Authz:** None — runs as the workflow file on the default branch.
- **Approve gate:** N/A.
- **Runs:** On cron; **disabled after 60 days of repo inactivity**[^events-docs].
- **Concurrency:** Skewed in gh-aw via fuzzy schedules to avoid thundering herds.
- **Forks:** Schedules in forks are disabled by default.
- **Unintended:** Long-running schedules + `stop-after:` mismatch can leave a workflow active longer than intended after a recompile.

## [`push`](../triggers/push.md)
- **Authz:** Push access required.
- **Approve gate:** N/A.
- **Runs:** On every matching push; can be filtered by `branches`, `tags`, `paths`.
- **Concurrency:** Use `${{ github.ref }}` group + `cancel-in-progress: true` for CI.
- **Forks:** Push events on forks fire on the fork; do not propagate.

## [`pull_request`](../triggers/pull-request.md)
- **Authz:** Anyone who can open a PR (incl. fork contributors).
- **Approve gate:** YES for first-time / outside contributors per org policy[^approval-policy].
- **Runs:** On the **PR's merge ref** with **read-only `GITHUB_TOKEN`** and **no secrets** for fork PRs[^token-permissions].
- **Concurrency:** `${{ github.workflow }}-${{ github.event.pull_request.number }}` with cancel-in-progress is the standard CI pattern.
- **Forks:** Allowed; this is the *safe* PR trigger.
- **Unintended:** None major — read-only token + no secrets is the safety net.

## [`pull_request_target`](../triggers/pull-request-target.md) 🛑 ⚠️ ⚠️ ⚠️
- **Authz:** Same actors as `pull_request`.
- **Approve gate:** Same as `pull_request`.
- **Runs:** On the **base ref** with **full `GITHUB_TOKEN` and access to all secrets**.
- **Concurrency:** Same patterns.
- **Forks:** Allowed — but you must **never check out the PR's head SHA** or run any code from it[^pwn-requests]. Safe uses: labeling, commenting, dependency review on the diff metadata.
- **Unintended:** 🛑 Most-exploited GitHub Actions vulnerability class — "pwn requests." Combined with [Authorization, Roles, and Read-Only Contributors](../chapters/authorization-and-roles.md): a fork contributor with **no repo access at all** can induce the workflow to execute arbitrary writes (comment, label, push to base, merge a different PR, etc.) using upstream secrets. **Always pair with `on.roles:` (default) and a fork-head guard.**

## [`issues`](../triggers/issue.md)
- **Authz:** Read access (anyone can open issues; only own issues can be closed/edited as a reader).
- **Approve gate:** N/A.
- **Runs:** Always.
- **Concurrency:** Group on `${{ github.event.issue.number }}` to serialize per-issue work.
- **Forks:** N/A (issues live on the upstream).
- **Unintended:** `issue.body`/`issue.title` are user-controlled — use gh-aw's `steps.sanitized.outputs.text` rather than raw `${{ github.event.issue.body }}` to avoid prompt-injection / template-injection[^command-doc].

## [`issue_comment`](../triggers/comment-and-slash-command.md) 🛑 ⚠️
- **Authz:** Read.
- **Approve gate:** N/A — even on forks.
- **Runs:** On every comment on issues *and* PRs; distinguish via `github.event.issue.pull_request != null`.
- **Concurrency:** Group on `${{ github.event.issue.number }}` — but be aware of the [non-matching-cancels-matching race](../chapters/concurrency-and-races.md).
- **Forks:** Comments on PRs from forks fire on the upstream and run with **upstream's secrets/token**.
- **Unintended:** 🛑 A PR-author who is a fork contributor (with **no** repo access) can comment on their own PR and trigger `issue_comment` workflows that run with upstream secrets. This is the second pwn-requests vector. See [Authorization, Roles, and Read-Only Contributors](../chapters/authorization-and-roles.md) for the full capability matrix. Defenses (in order): keep `on.roles:` at default (excludes anonymous fork actors), add `skip-roles:` for fine-grained blocks, never check out PR head SHA in the same job, audit `safe-outputs:` for the actual blast radius.

## [`pull_request_review`](../triggers/pull-request-review.md) / [`pull_request_review_comment`](../triggers/pull-request-review-comment.md) 🛑 ⚠️
- Same caveats as `issue_comment`. Reviews fire even from fork contributors with upstream secrets — same 🛑 pwn vector applies.

## [`discussion`](../triggers/discussion.md) / [`discussion_comment`](../triggers/discussion-comment.md)
- **Authz:** Read.
- **Approve gate:** N/A.
- **Runs:** On the upstream repo only.
- **Forks:** N/A.
- **Unintended:** Same content-sanitization concerns; lower stakes because typically no PR/code context.

## [`workflow_run`](../triggers/workflow-run.md) 🛑 ⚠️
- **Authz:** Indirect (whoever can trigger the upstream workflow).
- **Approve gate:** N/A.
- **Runs:** With **base repo's secrets and write token**, on the **default branch** of the workflow file.
- **Concurrency:** Standard.
- **Forks:** Fires for runs originating from fork PRs — this is the canonical "smuggle untrusted artifacts back into a trusted context" trigger[^pwn-requests].
- **Unintended:** Downloading and operating on artifacts from the triggering run is the classic vulnerability. gh-aw enforces fork/repository-ID checks and requires `branches:` to be set[^triggers-doc].

## `repository_dispatch`
- **Authz:** PAT with `repo` scope.
- **Approve gate:** N/A.
- **Runs:** With base secrets.
- **Concurrency:** Standard.
- **Forks:** N/A.
- **Unintended:** Anyone with the PAT can fire arbitrary `event_type` strings.

## [`workflow_call`](../triggers/workflow-call.md)
- **Authz:** Caller's authz.
- Effectively a function call between workflows — security inherits caller.

## [`slash_command:`](../triggers/comment-and-slash-command.md) (gh-aw) 🛑 ⚠️
- **Authz:** Inherits the underlying event (read for comments/issues/discussions, fork rules for PRs). **gh-aw auto-injects `on.roles: [admin, maintainer, write]`** — read/triage actors fire the workflow but the agent step is skipped unless `on.roles:` is broadened or set to `all`. **Setting `on.roles: all` opens every cell of the [capability matrix](../chapters/authorization-and-roles.md) to anyone who can post a comment.**
- **Approve gate:** Inherits — comment-only commands escape the gate; PR-body commands hit it.
- **Runs:** On *every* event in the (default-or-narrowed) `events:` list. Activation job filters.
- **Concurrency:** See the [non-matching-cancels-matching race](../chapters/concurrency-and-races.md) — broad event subscription + shared concurrency group is the pathology.
- **Forks:** Same as comments/PRs — comment-on-own-fork-PR is the dangerous path 🛑.
- **Unintended:** Aliased commands (`name: [a, b, c]`) multiply the surface; first-word matching means `` `/deploy` `` (in markdown code) does not trigger but `/deploy now` does.

## [`label_command:`](../triggers/labeled-and-label-command.md) (gh-aw)
- **Authz:** Triage+ (label application requires triage role).
- **Approve gate:** N/A.
- **Runs:** When a matching label is applied; auto-removes label so the command is one-shot.
- **Concurrency:** Group on `${{ github.event.issue.number }}`.
- **Forks:** N/A — labels are upstream.
- **Unintended:** `remove_label: false` turns it into a state marker rather than a command — different semantics. Triage role is the de-facto authz boundary.

📚 See [Appendix B: Footnotes](footnotes.md) for source citations.

---

<nav>

[← Previous: Authorization, Roles, and Read-Only Contributors](../chapters/authorization-and-roles.md) | [Table of Contents](../README.md) | [Next: Appendix B — Footnotes →](footnotes.md)

</nav>
