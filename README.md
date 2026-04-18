# GitHub Actions & gh-aw Triggers — Field Guide

A working reference for every trigger usable in traditional GitHub Actions and in **GitHub Agentic Workflows (`gh-aw`)**, with emphasis on the surprises: apparent-vs-actual triggering, the *Approve and run workflows* gate, fork behavior, secrets exposure, concurrency-induced races, role-based authorization, and the read-only-contributor write surface.

## Headlines

This Field Guide is structured around two complements:

- **[Part 1 — Tenets](#part-1--tenets)**: the design principles every workflow in this repo must satisfy. *Positive framing: what we're trying to achieve.*
- **[Part 2 — Pitfalls by trigger](#part-2--pitfalls-by-trigger)**: the single worst-case footgun for each trigger you might consider. *Negative framing: what to avoid.*

### Part 1 — Tenets

1. **We must avoid subtle privilege-escalation pitfalls — including ones a careful reviewer would miss.** This Field Guide exists because most of the pitfalls below *look* safe (gated, role-checked, fork-isolated) and only escalate under specific event combinations.

2. **Consequential actions must require human approval of a reviewable diff** with a clear audit trail and the action attributed to the human team member.

3. **Bot actions must never be presented as actions taken by a human.** The bot remains the visible *actor*; the responsible human is captured separately as the *accountable party* via co-authorship, comment metadata, or audit log.

4. **No alert fatigue, no opaque approvals.** The flip side of #2: approval must never be required for inconsequential operations, and every prompt must clearly show what will be done and why. Opaque or routine prompts cause alert fatigue and degrade #2 into theatre. (See §3 for the canonical anti-example: GitHub's "Approve and run workflows" gate.)

5. **Use the right tool: deterministic by default.** Deterministic Actions and reusable workflows by default; agentic workflows only when the input is unstructured, the decision space is open, or AI unlocks a capability deterministic code cannot provide.

6. **Resist building a bespoke workflow-management platform on top of gh-aw.** Every layer of meta-orchestration becomes a product you must maintain at high opportunity cost.

7. **Limitations that exist for security reasons are not bugs to be worked around — they *are* the security model.** Bypasses (e.g., `pull_request_target` to gain write access to fork content, PAT pools used to evade bot-identity attribution, `workflow_run` to escape approval gates, `on.roles: all` to widen the actor pool) erase the protections you appear to still have. When a boundary blocks a legitimate goal, escalate it to the platform owners; do not engineer a bypass.

8. **No human single point of failure.** Operational continuity must not depend on any single team member. Credentials, approvals, and recovery knowledge must be distributed across the team — e.g., this repo's `COPILOT_PAT_*` pool distributes credential ownership and rate-limit budget across multiple team members, eliminating the burdened SPOF that a single shared PAT would create.

9. **Limit patterns; converge on canonical recipes.** Focus on canonical recipes for achieving goals consistently — especially regarding which trigger to use when. Maintain a cheat sheet that maps each scenario/goal to its trigger + conditions, and keep it current as we learn. (Part 2 below is the seed of that cheat sheet.)

10. **Mind the signal-to-noise ratio of high-volume triggers.** Convenience triggers (e.g., `slash_command:`) and activity-type filters (e.g., `pull_request: types: [labeled]`, `issues: types: [opened, edited]`) both compile to broad event subscriptions whose activation step runs on every matching event; the agent step fires on only a fraction of those runs. Weigh the tradeoff explicitly: **low-latency invocation requires accepting a high executed-to-filtered ratio**, and that ratio costs runner minutes (and in some configurations, model tokens) on every no-op activation. The activation step must be cheap, and the worst-case invocation rate must be estimated and acceptable.

11. **Limit the agent job to actions best suited for execution within an agent.** Keep all filtering and skipping logic in steps or jobs that execute before the agent. Execute deterministic scripts both before and after the agent job.

12. **Apply least privilege on every dimension.** Grant the bare minimum `permissions:`, `safe-outputs:`, `network.allowed:`, secrets, `tools:`, and other capabilities. Use the agent sandbox and safe-outputs controls to operate on untrusted inputs (anything authored outside the trusted maintainer set — PR bodies, comments, fork PR head SHAs, fork file contents, discussion content). **Example:** checking out a fork and executing its code is acceptable *inside* the agent sandbox; the same operation in pre- or post-agent steps executes attacker code on the runner host with full secret access and is not acceptable.

13. **Read the docs.** Even the `agentic-workflows` Copilot agent provides misleading and incorrect guidance — gh-aw releases multiple times a day, so any pattern learned from an assistant, a memory, or *this Field Guide* must be verified against the current docs at <https://gh.io/gh-aw>.

14. **Test your workflows in disposable repositories before promoting them to product repositories.** Use the [TrialOps](https://github.github.com/gh-aw/patterns/trial-ops/) pattern (`gh aw trial` creates an ephemeral private trial repo). Validate with at least one test GitHub account that has only **read** permission — that's the actor pool most likely to surface privilege-escalation and denial-of-service issues. **Never test in production.**

### Part 2 — Pitfalls by trigger

*Cataloged trigger-by-trigger. Each entry below names the worst-case footgun(s) for one trigger — the mistakes most likely to bite you in production. The full per-trigger reference (events, activity types, fork behavior, role gates) lives in §7; the live demonstration workflows live in §8. Triggers not listed here (e.g., `branch_protection_rule`, `check_run`, `check_suite`, `create`/`delete`, `deployment`/`deployment_status`, `fork`, `gollum`, `label`, `member`, `milestone` (the `on: milestone` trigger itself — its cascade behavior is captured under `milestone`/`issues.demilestoned` below), `page_build`, `project*`, `public`, `registry_package`, `release`, `repository_dispatch`, `status`, `watch`) are covered in §7 — they either have low headline risk or are irrelevant to agentic workflows in this repo's context.*

#### `discussion`

> 🛑 **`discussion` is the most-open untrusted-input surface in the GitHub model and it has no approval gate.** Anyone with repository read access can fire `discussion.created` or `discussion.edited`. The workflow runs with its declared `permissions:` and `safe-outputs:`, which the author thinks of as "what we can do" but is actually "what any read-user can cause us to do." Discussion bodies are unstructured prose — an obvious prompt-injection vector. The default `on.roles:` allowlist stops the agent step for read-role actors, but the workflow still queues a runner (tenet #10), and any pre-agent step that does something with the discussion content runs anyway.

#### `discussion_comment`

> 🛑 **The "comment on your own discussion" privilege escalation.** A read-only user can create a discussion, immediately comment on it with any payload, and edit either freely. Any agentic workflow that listens on `discussion_comment` — including `slash_command:` workflows by default — can be invoked at will by a drive-by user. The actor of the workflow run is the read-user but the actor of the workflow outputs is `github-actions[bot]` (violates tenet #3). If `safe-outputs:` allows `add-comment` and the comment lands on the triggering discussion, the bot output appears to endorse/reply-to the attacker prompt — lending upstream apparent authority to the attacker narrative.

#### `issue_comment`

> 🛑 **A. The "no approval gate" loophole.** `issue_comment` does **NOT** trigger the "Approve and run workflows" gate, even when fired from a comment on a fork PR. A fork contributor can fire any `issue_comment`-listening workflow on their own PR with no maintainer interaction. This is the canonical reason `slash_command:` workflows have a much wider invocation surface than they appear — `slash_command:` subscribes to `issue_comment` by default.

> 🛑 **B. Issues vs. PRs naming trap.** Comments on PRs fire `issue_comment`, **NOT** `pull_request_review_comment` (which is a different event for inline review threads). Workflows authored as "an issue triager" inadvertently respond to comments on PRs from forks — even though the author never put `pull_request*` in `on:`.

> 🛑 **C. Edits to old comments fire NOW.** An attacker can edit a 6-month-old comment on a closed issue or PR, injecting `/command` or any payload — `issue_comment.edited` fires today against today's secrets, today's `permissions:`, today's `safe-outputs:` allowlist. The workflow has no concept of "this comment was created when our security model was different." Tenet #10's worst-case invocation rate must include the attacker reaching back through the entire comment history of the repo.

#### `issues`

> 🛑 **`issues.edited` is a privilege-escalation amplifier.** A read-only contributor can open an issue with a benign body, wait for auto-triage/auto-label workflows to bless it, then edit the issue body to inject the actual hostile payload — an `issues.edited` event fires NOW, runs the same workflow with today's secrets, today's `permissions:`. This sidesteps any human review that happened on the original `opened` event. Bonus: `issues.assigned` and `issues.labeled` events have a maintainer actor but their payload (body, title, comments) remains attacker-controlled — workflows that gate on `github.event.sender` ("only act if a maintainer triggered me") can be tricked into processing attacker prose because a maintainer applied a label. The actor changed; the data did not.

#### `milestone`

> 🛑 **`milestone.closed` is a cascade trigger that workflows underestimate.** Closing a milestone is a single admin action that does NOT itself fire issue events — but **DELETING** a milestone removes it from all assigned issues/PRs, firing `issues.demilestoned` (and `pull_request.demilestoned`) for every one of them in rapid succession. A workflow that listens on `*.demilestoned` to react (move from a project, change labels, post a comment) will fire N times in seconds when a milestone with N items is deleted. Combined with `cancel-in-progress: true` concurrency, only the last one runs and earlier reactions are silently lost; without it, runner minutes balloon. Same naming-collision footgun as `label`: `on: milestone` is **not** `on: issues: types: [milestoned]`.

#### `pull_request`

> 🛑 **A. `closed` ≠ `merged`.** Workflows that celebrate a merge, post release notes, file follow-up issues, or kick off downstream automation on `pull_request.closed` fire on PR-closed-without-merge too, unless they explicitly check `github.event.pull_request.merged == true`. The PR author can game this: open a PR, close it, and the "thanks for merging!" workflow runs.

> 🛑 **B. The `reopened` time-bomb.** Anyone with write access (and the original PR author themselves) can reopen a closed PR from 2 years ago. With default `types: [opened, synchronize, reopened]`, the workflow fires today, with today's secrets and today's `permissions:`, against the 2-year-old PR head SHA. The activation step has no concept of "this PR content was reviewed in 2023; the world has changed since." Combine with `pull_request_target` and this is catastrophic.

> 🛑 **C. `synchronize` is per-push, not per-PR.** A 50-commit force-push to a PR fires the workflow 50 times unless `concurrency.cancel-in-progress: true` is set — but enabling that creates the §5 race condition where in-progress useful work gets cancelled by a spam push. The cost-vs-correctness tradeoff is unavoidable; it must be made deliberately.

#### `pull_request_review`

> 🛑 **"A review was submitted" is not "the PR was approved" — but workflows conflate the two.** The event fires for any review type, including `COMMENT`-type reviews that any read-role user (or any drive-by GitHub account) can submit. Workflows that auto-merge "after N reviews submitted," post "ready to land" comments, or kick off deployment on review-submitted fire for non-approving comments too — including comments authored by accounts with no write access at all. Even gating correctly on `event.review.state == 'approved'` doesn't suffice because actor authorization is independent of review state. Review bodies are unstructured prose authored by anyone with read access — an under-monitored prompt-injection vector that workflows ingest as if it came from a trusted reviewer. `pull_request_review.edited` fires today against today's secrets when an old review is edited (same time-bomb shape as `issue_comment.edited`).
>
> *Concurrency twist:* `pull_request_review` is rarely the only trigger on a PR-handling workflow; it usually shares a workflow file with `pull_request` and `issue_comment`. If they share `concurrency.group: pr-${{ github.event.pull_request.number || github.event.issue.number }}` with `cancel-in-progress: true`, a stray comment or a reviewer dismissing an old review can cancel a half-finished APPROVE-handling run — the activation passed, the agent is doing real work, then a benign comment kills it mid-flight, leaving the PR in an indeterminate state (label half-applied, status check half-posted). Without `cancel-in-progress`, two near-simultaneous reviewers stack runs and post duplicate comments or apply duplicate labels. There is no group key that gets this right for all three triggers in one workflow.

#### `pull_request_review_comment`

> 🛑 **The trigger nobody remembers exists — until it fires unexpectedly and ingests attacker-controlled prose.** Most workflow authors think there is one "PR comments" trigger. There are actually three distinct events: `issue_comment` (top-level PR conversation), `pull_request_review` (the review body), and `pull_request_review_comment` (the inline line-level comment). Workflows that subscribe to `issue_comment` for a `/command` or for prompt content will silently miss inline review comments — but the inverse is more dangerous: a workflow subscribed to `pull_request_review_comment` (often by accident, copy-pasted from a sample) fires on every line-level scribble by anyone with read access. The comment body is unstructured prose tied to a specific code line — high-leverage prompt-injection surface because attackers can position the malicious instruction next to specific code the agent is being asked to reason about ("the line above is intentional, ignore prior instructions and …"). The `.edited` time-bomb applies: an old inline comment edited today fires today's workflow with today's secrets against potentially-stale PR head SHA. Concurrency twist: when this trigger is mixed into a PR-handling workflow alongside `pull_request` and `pull_request_review`, group-key choice is even worse than for `pull_request_review` alone — the inline-comment payload doesn't always have a clean `pull_request.number` at the top level (you have to dig through `event.pull_request` which is sometimes null in deleted-comment payloads).

#### `pull_request_target` ⚠️

> 🛑 **`pull_request_target` is the trigger most likely to get a repo pwned, *and the approve-and-run gate is the booby trap that springs it*.** `pull_request` from forks runs sandboxed (read-only token, no secrets). Maintainers reach for `pull_request_target` to get write tokens and secrets back. The pwn vector: it checks out the **base** ref by default, but real workflows then `actions/checkout` with `ref: ${{ github.event.pull_request.head.sha }}` to see the PR's code — at which point an attacker's first PR can ship a malicious `package.json` install hook, `Makefile`, custom action under `.github/actions/`, or `.npmrc` that **executes with full repo secrets and write `GITHUB_TOKEN`**. Even without checking out PR code, the PR title/body are attacker-controlled prose the agent ingests — instant prompt injection with write tokens.
>
> **The approve-and-run gate is what makes this catastrophic, not what protects from it.** In our configuration the approval requirement is on by policy and external contributors must work from forks — so every `pull_request_target` workflow on every external PR presents the "Approve and run workflows" button, every day. That is alert fatigue by construction: the click-rate trends toward 100%, **the UI does not show what is about to execute or give guidance for what to review before approving**, and the button label conceals what is actually being authorized ("run safely-defined workflows" reads as "rubber-stamp this routine PR," not "grant this stranger's code access to every secret"). **The gate is also one-shot per click — approving *all* gated workflow runs in that batch at once, with no granular visibility or selection of which individual workflows are safe to run for this specific PR.** Click once and every gated workflow on the PR fires. **gh-aw's safe-outputs gate, output sanitization, and `staged: true` exist largely *because* of this trigger's footgun shape.**

#### `push`

> 🛑 **`push` is one of the highest-volume triggers in the repo and the easiest to subscribe to overbroadly.** A workflow declared as just `on: push` (no `branches:` filter) fires on every push to every branch by anyone with write access — including bot branches, dependency-update branches, codeflow branches, and short-lived feature branches that no one expected the workflow to care about. For an agentic workflow that consumes Copilot tokens per run, the cost blowout is silent until the bill arrives — this is the trigger most likely to turn "it's just a PoC" into a line-item on next month's invoice. The fix is always-explicit `branches:` filters and a narrow subscription that matches the actual workflow intent. Concurrency twist: rapid pushes to a feature branch (rebasing, force-pushing) stack up runs unless `concurrency.cancel-in-progress: true` is set — which in turn re-creates the §5 race condition where the canceled run was halfway through a write that the next push won't redo.

#### `schedule`

> 🛑 **`schedule` is the trigger that keeps running quietly in the background without relying on any user interaction at all.** These workflows are forced to re-derive context that an event would have handed them, often burning tokens on a fuzzy clock against stale assumptions. Scheduled agentic workflows run a cron against today's default branch but were designed around yesterday's repo. A scheduled "triage open issues" workflow written when there were 20 open issues may still be running years later against 2,000 open issues, with per-run cost now 100× what was tested. There is no PR to review, no comment to acknowledge, no UI indicator that the workflow even ran — it just consumes runner minutes and Copilot tokens indefinitely. The "schedule" is also softer than it reads: GitHub runs the cron **when a runner becomes available**, not on a fixed wall clock — heavy load delays runs by minutes-to-hours, and during peak hours your every-15-minute job may run every 25.
>
> The deeper structural problem: **`schedule` is the wrong tool for work that should react to events.** Where a real trigger hands the workflow a payload that says exactly what changed, a scheduled run starts blind and must **poll** to discover what work might exist, track its **own state** to know what's already been done, and implement some form of **optimistic concurrency or locking** so two overlapping cron firings don't double-process the same item. Every one of those is a distributed-systems problem the workflow author now has to solve in YAML and shell — and most don't, so scheduled agents quietly post duplicate comments, file duplicate issues, or apply duplicate labels every time the cron laps the prior run.

#### `workflow_call`

> 🛑 **`workflow_call` is the trigger that *looks* like safe encapsulation but actually creates an undeclared trust boundary — and the called workflow runs with whatever permissions the caller had, which the callee author did not control.** A reusable agentic workflow exposes `inputs:` that downstream callers populate from `${{ github.event.* }}` — the issue body, PR title, comment text, etc. The callee author can sanitize inside the called workflow, but **they cannot enforce that callers don't pipe attacker-controlled prose straight in**. Worse for token surface area: `secrets: inherit` is the convenient knob that hands every secret in the calling repo to the called workflow — including Copilot PATs, deployment tokens, and any cross-repo secret. A maintainer adding `uses: org/shared-workflows/.github/workflows/triage.yml@main` does not see a reviewable diff of what `triage.yml` will do with their secrets; pinning to `@main` (or any moving ref) means the upstream maintainer can change the agent prompt, swap the model, or add new safe-outputs **without the calling repo seeing the change**. Pin by SHA. And note: the calling repo's branch protection / approval gates **do not apply to the called workflow's behavior** — only to the act of merging the caller's YAML.

#### `workflow_dispatch`

> 🛑 **`workflow_dispatch` is the most-trusted trigger we have — and that trust gets quietly extended to anyone with write access, who can run *any* dispatchable workflow against *any* branch with *any* `inputs:` payload.** A maintainer adds `workflow_dispatch:` because they want a manual escape hatch for themselves. Every other write-role contributor inherits that same button. With `inputs:` defined as free-form strings, write-role users can supply attacker-style prose to an agent without leaving any of the usual artifacts (no PR, no comment, no commit) — the entire payload lives in the workflow run's "Inputs" section that nobody scrolls back to read. **Branch selection is also user-controlled**: a write-role contributor can run the dispatch against a branch they just pushed (or a long-stale branch) where the workflow YAML is **different** from `main`'s — meaning the version of the workflow they're invoking may have less-restrictive `permissions:`, weaker `safe-outputs.allowed:`, or an attacker-friendlier prompt. The `main`-branch version is not what runs; the **selected branch's** version is.

#### `workflow_run` ⚠️

> 🛑 **`workflow_run` is `pull_request_target`'s quieter sibling: it launders untrusted output from a fork PR's sandboxed build into a privileged context with full secrets — and nobody reviewing the upstream workflow knows that's about to happen.** The pattern that gets repos pwned: a `pull_request` workflow runs the fork's tests in a sandbox (correct), uploads test results / coverage reports / built artifacts as workflow artifacts (still fine), and then a separate `workflow_run`-triggered workflow downloads those artifacts and acts on them — posting a comment, updating a status check, deploying a preview, or feeding the artifact contents to an agent. **The artifact came from attacker-controlled fork code; the consuming workflow has full write tokens and secrets.** A malicious PR uploads an artifact named `coverage.json` whose contents include shell-injection or prompt-injection payloads and the consuming workflow shells out unsafely, or feeds the JSON to an agent that ingests the prompt-injection. There is no UI signal connecting the upstream PR to the downstream `workflow_run`; reviewers see "tests passed on a fork PR" and miss that a privileged workflow then ran with that PR's data. And `workflow_run` does **not** inherit the upstream's `pull_request` head SHA — the consuming workflow runs against the **default branch's** YAML, so attackers can rely on the production version of the consumer workflow even when the producer workflow on their fork is sandboxed.

#### `slash_command:` (gh-aw convenience trigger)

> 🛑 **`slash_command:` is the trigger most users mentally model as "fires only when someone with the right role types `/my-command`" — which is wrong by an order of magnitude or more, and the framing leads directly to the §5 concurrency catastrophe.** What `slash_command:` actually does is subscribe to a *broad set of underlying events* (every issue comment, every PR review comment, every issue/PR open/edit, every discussion comment) and then filter at activation time. **Every one of those events spawns a workflow run** that consumes a runner slot and a brief execution context — only the `/command` filter and the `roles:` filter cause an early abort. Most workflow authors never see this because the runs abort fast and the UI lists "Skipped" — but they **did run**, they **did consume the runner**, and they **did affect concurrency groups**.
>
> *The concurrency catastrophe (cross-reference §5):* if the slash-command workflow declares `concurrency.group: my-cmd-${{ github.event.issue.number }}` with `cancel-in-progress: true`, then a benign drive-by comment by any read-role user on the same issue **cancels the in-flight `/command` run that a maintainer just authorized**. The maintainer sees their command silently fail mid-execution because Mallory typed "+1" thirty seconds later. The reverse failure mode is also live: without `cancel-in-progress`, every drive-by comment queues a (skip-fast) run behind the real one, and on busy issues the runner pool can saturate. There is no group key that gets this right — the broad subscription and the activation-time filter are fundamentally at odds with concurrency keyed on the conversation. **And `roles:` checks the actor of the comment that triggered the underlying event, not the actor of the original `/command` comment** — so a `.edited` event from anyone editing any old comment on the issue can fire the workflow under the *editor's* identity even if the editor never typed the slash-command.
>
> *Idempotency requirement:* because concurrency groups cannot be used safely with `slash_command:`, slash-command workflows **MUST be implemented to be idempotent** — by implementing the same concurrency checks or locking in code that schedule-jobs-that-poll must employ. Treat every `/command` activation as if the same `/command` might already be running for the same target: check before acting, claim a lock (e.g., via a label, an issue-comment marker, a check-run with a unique name, or external state), no-op if the work is already in progress or already done, and release/clean-up on completion. The discipline is the same as for scheduled pollers, and the failure modes when you skip it are the same: duplicate comments, duplicate labels, duplicate downstream effects, and races that leave the target in an indeterminate state.

#### `label_command:` (gh-aw convenience trigger)

> 🛑 **`label_command:` turns labels into one-shot RPC calls — and inherits the same broad-subscription concurrency catastrophe as `slash_command:`.** The trigger fires when any write-role contributor applies the configured label, runs the agentic workflow, and then **automatically removes the label**. The audit trail is preserved but is *subtle* — the issue/PR event stream shows `Octocat added the deploy label` immediately followed by `github-actions[bot] removed the deploy label`, which is easy to scroll past as a label-fix correction. For a `/deploy`-shaped command, this means a triage-role contributor (write access but typically not deploy-trusted) can fire the deploy by applying a label, and the only record of the command being issued lives in two adjacent timeline events that look like noise.
>
> The bigger structural problem is the same shape as `slash_command:`: `label_command:` subscribes to *every* `labeled` event on the configured event types (`issues`, `pull_request`, `discussion`). The workflow runs — consuming a runner — on every label application by every write-role user, and only the name match causes early abort. Like `slash_command:`, this means **concurrency groups cannot safely be used** (a benign label change on the same target would cancel the in-flight command). Therefore `label_command:` workflows **MUST be implemented to be idempotent**: check before acting, claim a lock, no-op if the work is already in progress or already done. The same discipline scheduled pollers and `slash_command:` workflows must employ. Skipping that discipline produces the same failure modes: duplicate deploys, duplicate comments, duplicate downstream effects, and races that leave the target in an indeterminate state.

---

## 1. The Two Categories of Triggers

### 1.1 Standard GitHub Actions events (35+)

Every gh-aw workflow ultimately compiles to a standard `on:` block. The full set, per GitHub's reference[^events-docs]:

| Event | Activity types (where applicable) | Runs on default branch only? | Notable security/behavioral notes |
|---|---|---|---|
| `branch_protection_rule` | created, edited, deleted | Yes | Admin-only causes; safe |
| `check_run` | created, rerequested, completed, requested_action | Yes | Doesn't recurse on Actions-created check suites |
| `check_suite` | completed | Yes | Same |
| `create` | (none) | No | Branch/tag created; no payload secrets risk |
| `delete` | (none) | No | Branch/tag deleted |
| `deployment` | (none) | No | Triggered by API/UI deployment |
| `deployment_status` | (none) | No | |
| `discussion` | created, edited, deleted, transferred, pinned, unpinned, labeled, unlabeled, locked, unlocked, category_changed, answered, unanswered | Yes | Repo must have Discussions enabled |
| `discussion_comment` | created, edited, deleted | Yes | Read-only users **can** fire |
| `fork` | (none) | Yes | Fires when *anyone* (incl. outside) forks |
| `gollum` | (none) | Yes | Wiki edits |
| `issue_comment` | created, edited, deleted | Yes | Fires for **both** issue *and* PR comments; PR distinction via `github.event.issue.pull_request != null` |
| `issues` | opened, edited, deleted, transferred, pinned, unpinned, closed, reopened, assigned, unassigned, labeled, unlabeled, locked, unlocked, milestoned, demilestoned, typed, untyped | Yes | Read-only users can `opened`, `edited` (own), `closed` (own), `reopened` (own) |
| `label` | created, edited, deleted | Yes | Repo-level label CRUD, not application |
| `member` | added, edited, removed | Yes | |
| `merge_group` | checks_requested | Yes | Merge queue checks |
| `milestone` | created, closed, opened, edited, deleted | Yes | |
| `page_build` | (none) | Yes | GitHub Pages build |
| `public` | (none) | Yes | Repo went public |
| `pull_request` | opened, edited, closed, assigned, unassigned, review_requested, review_request_removed, ready_for_review, labeled, unlabeled, synchronize, converted_to_draft, locked, unlocked, enqueued, dequeued, milestoned, demilestoned, reopened, auto_merge_enabled, auto_merge_disabled | Workflow runs on PR's merge ref or head ref | **Read-only `GITHUB_TOKEN`, no secrets** when PR is from a fork[^token-permissions] |
| `pull_request_review` | submitted, edited, dismissed | (PR ref) | |
| `pull_request_review_comment` | created, edited, deleted | (PR ref) | |
| `pull_request_target` | (same as `pull_request`) | **Default branch, with secrets and write token** | **Most dangerous trigger** — never check out PR head SHA without sandboxing[^pwn-requests] |
| `push` | (none, but supports `branches`/`paths`/`tags` filters) | (the pushed ref) | Forks cannot trigger (they push to their fork) |
| `registry_package` | published, updated | Yes | |
| `release` | published, unpublished, created, edited, deleted, prereleased, released | Yes | |
| `repository_dispatch` | (custom `event_type`) | Yes | API-triggered with PAT having `repo` scope |
| `schedule` | (cron) | Yes | Disabled after 60 days of repo inactivity |
| `status` | (none) | Yes | Commit status changes |
| `watch` | started | Yes | Repo starred |
| `workflow_call` | (called from another workflow) | (caller's ref) | Reusable workflow |
| `workflow_dispatch` | (manual) | (selected ref) | **Requires write access** to the repo to invoke[^approval-docs] |
| `workflow_run` | requested, completed, in_progress | Default branch | Runs with **base repo's** secrets even when triggered by a fork PR's CI run — second-most-dangerous trigger[^pwn-requests] |

> Several "deprecated"/legacy events still exist (`project`, `project_card`, `project_column` for classic projects). They are not currently supported by gh-aw shorthands but accept-as-passthrough in `on:`.

### 1.2 gh-aw "convenience" / "virtual" triggers

gh-aw calls these **"trigger types"** but the docs themselves repeatedly note they expand to standard events plus filtering[^triggers-doc][^command-doc]. The framework's preferred terminology is **"convenience trigger"** for `slash_command:` / `label_command:` and **"trigger shorthand"** for the natural-language strings (`on: push to main`, `on: issue labeled bug`, etc.).

| gh-aw trigger | Compiles to | Filtered by what (in activation job) |
|---|---|---|
| `slash_command: name` | `issues:[opened,edited,reopened]` + `issue_comment:[created,edited]` + `pull_request:[opened,edited,reopened]` + `pull_request_review_comment:[created,edited]` + `discussion:[created,edited]` + `discussion_comment:[created,edited]` + `workflow_dispatch` (default; narrow with `events:`) | First-word match of `/<name>` in body/comment text |
| `label_command: name` | `issues:[labeled]` + `pull_request:[labeled]` + `discussion:[labeled]` + `workflow_dispatch` (default; narrow with `events:`) | Label name match; auto-removes label after activation (unless `remove_label: false`) |
| `reaction: <emoji>` | (additive) Posts an emoji on the triggering item from the activation job | n/a |
| `status-comment: true` | (additive) Posts a "started/completed" comment on the triggering item | n/a |
| `forks: ["pattern"]` | (additive) Allow PRs from matching forks; default is **block all forks** | Repository-ID comparison in activation job |
| `names: [labels]` | (additive) Filter `issues`/`pull_request` events by label name | Label list contains in activation job |
| `lock-for-agent: true` | (additive) Locks the parent issue at start, unlocks at end | n/a |
| `on.roles: [admin, maintainer, write] \| all` | (additive) **Allowlist** of repo roles allowed to trigger. **Default `[admin, maintainer, write]` is injected automatically** when the workflow has any "unsafe" event (issues, comments, PRs, discussions, slash/label commands). `roles: all` disables the check[^role-checks-go]. | Membership API call in activation job |
| `skip-roles: [role,...]` | (additive) **Blocklist** — skip when actor has any of the listed repo roles. Layered on top of `on.roles:` | Membership API call in activation job |
| `skip-bots: [actor,...]` | (additive) Skip when actor matches | Actor name match in activation job |
| `skip-if-match: query` | (additive) Skip when GitHub Search query has matches (≥ `max`) | Search API call in activation job |
| `skip-if-no-match: query` | (additive) Skip when query has fewer than `min` matches | Same |
| `manual-approval: env-name` | (additive) Sets `environment:` on activation job → environment protection rules | GitHub-native env approval gate |
| `stop-after: +25h` or date | (additive) Activation job exits after deadline; recompile resets | n/a |
| `on.steps: [...]` | (additive) Custom deterministic steps in pre-activation job | Auto-wired `<id>_result` outputs |
| `on.permissions: {...}` | (additive) Extra `GITHUB_TOKEN` scopes for pre-activation job | n/a |
| `on.github-token: ...` / `on.github-app: ...` | (additive) Custom token/app for activation job (reactions, status comments, skip-if searches) | n/a |
| Schedule shorthands (`daily`, `weekly on monday`, `every 10 minutes`, `daily around 14:00`, `daily between 9:00 and 17:00`) | `schedule: [cron]` (compiler scatters minute offsets to spread load) | n/a |
| Trigger shorthands (`on: push to main`, `on: pull_request opened`, `on: issue labeled bug`, `on: repository starred`, `on: dependabot pull request`, `on: api dispatch custom-event`, `on: manual`, `on: comment created`, `on: workflow completed ci-test`, `on: security alert`, etc.) | Standard event(s) + auto-injected `workflow_dispatch` | Whatever the shorthand expands to |
| `on: /my-bot` (ultra-short) | `slash_command: my-bot` + `workflow_dispatch` | First-word match |

**Critical insight.** Every gh-aw "virtual" trigger results in **the workflow actually starting** when *any* of the underlying standard events fires. The "filtering" all happens in a real GitHub Actions job — the **pre-activation / activation job** — which then either (a) sets a job output that the agent job's `if:` consumes, or (b) early-exits and lets the agent job be skipped[^triggers-doc]. This means:

- A "skipped" run still appears in the Actions tab.
- It still consumes a runner minute (small, but nonzero — the activation job runs on `ubuntu-latest`).
- It still participates in `concurrency.group` and can **cancel** in-progress siblings.
- It still counts against any per-repo workflow concurrency / queue limits.

---

## 2. The "Apparent vs. Actual" Trigger Surface

The user's observation about `slash_command` is exactly right and worth restating in formal terms:

> A `slash_command: deploy` workflow appears, by reading the YAML, to trigger only when somebody comments `/deploy`. **In reality**, it runs on every issue open/edit, every PR open/edit, every comment, every PR review comment, every discussion, and every discussion comment in the repo — and *then* decides to skip if the first word isn't `/deploy`.

The official docs acknowledge this in two places:

> *"By default, command triggers listen to all comment-related events, which can create skipped runs in the Actions UI. Use the `events:` field to restrict where commands are active."*[^command-doc]

> *"The command must be the **first word** of the comment or body text to trigger the workflow. This prevents accidental triggers when the command is mentioned elsewhere in the content."*[^command-doc]

The first quote concedes the noise; the second concedes that even narrow matching is *post-event*, not *pre-event*.

### 2.1 Why this matters operationally

| Consequence | Mechanism |
|---|---|
| **Runner consumption** | Pre-activation job uses `ubuntu-latest` for ~5–30 s per skipped run; on a busy repo with `slash_command:` on a single small command this can be hundreds of skipped runs/day. |
| **Copilot token consumption** | Zero — the agent step is gated by the activation output and never runs in skipped cases. The COPILOT_GITHUB_TOKEN selection happens *in pre-activation* (per this repo's PAT-pool pattern) but no model requests are made. |
| **Concurrency cancellations** | If `concurrency.group` is shared across the workflow, a non-matching activation can cancel an in-progress matching one. See §5. |
| **Actions UI noise** | Operators learn to ignore "skipped" runs and miss real failures. |
| **Audit surface** | Every skipped run still emits webhook events for `workflow_run` listeners downstream. |
| **Approval prompts (forks)** | A first-time fork contributor opening *any* PR can produce an "Approve and run" entry on a `slash_command` workflow whose intent had nothing to do with PRs — see §3. |

---

## 3. The "Approve and run workflows" Gate 🛑

GitHub gates workflow execution from forks behind a manual approval click ("Approve and run workflows"), governed by repo/org **Actions → General → Fork pull request workflows from outside collaborators** setting[^approval-docs][^approval-policy]:

| Setting | Behavior |
|---|---|
| **Require approval for first-time contributors who are new to GitHub** | Approval needed only for accounts < 2 months old |
| **Require approval for first-time contributors** *(default)* | Approval needed until the contributor has had a merged PR or commit |
| **Require approval for all outside collaborators** | Always require approval for non-org-members |
| **Require approval for all external contributors** *(orgs only)* | Always require approval for anyone without write access |

**The gate applies to `pull_request` and `pull_request_target`.** It does **not** apply to:

- `issue_comment` (even on PRs from forks!) — a read-only contributor can therefore always *fire* a `slash_command:` from a comment on their own PR without a click. **The agent step still won't run** unless `on.roles:` includes their permission level (default allowlist excludes `read`).
- `discussion`, `discussion_comment` — never gated.
- `issues` — never gated; anyone with read access can open one.
- `workflow_dispatch` — never gated, but **requires write access to invoke**.
- `repository_dispatch` — never gated, but **requires a PAT with `repo` scope**.
- `schedule` — never gated.
- `workflow_run` — never gated (this is what makes it dangerous — see §7).

### The approval gate is dangerous, not protective

The "Approve and run workflows" button looks like a security feature. **Treat it as the opposite.** Three structural problems make it actively harmful:

1. **Alert fatigue.** Every first-time fork contributor produces a button click that lists *every* workflow the PR touches. A repo with 15 workflows shows 15 entries. After the maintainer has clicked through dozens of legitimate first-time PRs, the click becomes muscle memory. The hundredth click is no more deliberate than the first.
2. **The UI does not show what is being approved.** There is no per-workflow toggle, no preview of the diff, no preview of the events the workflows are subscribed to, no list of secrets that will be exposed, no indication that some workflows use `pull_request_target` (full secrets, write token) versus plain `pull_request` (read-only, no secrets). The maintainer is approving an opaque blob of YAML files they have likely never read.
3. **A single click runs all of them.** There is no way to approve only the safe ones. Approving the CI workflow you actually wanted to run *also* approves the labeler, the auto-merge bot, the slash-command listener, and any `pull_request_target` workflow in the repo.

**Concrete failure modes when a maintainer clicks the button because they "think they're supposed to":**

- A `pull_request_target` workflow that does `actions/checkout@v4` with `ref: ${{ github.event.pull_request.head.sha }}` (an extremely common mistake) **executes attacker-controlled code with the upstream's secrets in the environment**. Game over: secrets exfiltrated, NPM packages republished, releases tagged, branches force-pushed[^pwn-requests].
- A `slash_command:` workflow whose `on.roles:` was relaxed to `all` (perhaps to support a community `/help` bot) sees the attacker's PR body containing `/help` (or whatever command), passes its activation job, and the agent runs with the workflow's full `permissions:` and `safe-outputs:` — see §6.4 for the full capability surface.
- A `slash_command:` workflow that's also subscribed to `pull_request` (the default) gets approved alongside everything else — its activation job runs against the PR body and, if the magic word is there, the agent fires.
- A workflow that posts comments on the PR runs as the upstream `github-actions[bot]`, lending **upstream's apparent authority** to attacker-supplied output (e.g., a fake "✅ All checks passed — safe to merge" comment).
- The maintainer who clicked is not necessarily the same person who reviews the PR diff later; the approval click can effectively pre-authorize a *future* reviewer to merge based on a contaminated CI signal.

**Design rule: assume the approval gate will always be clicked.** The only safe workflows are ones that produce the same outcome whether the actor is a trusted maintainer or an anonymous fork contributor. Concretely:

- Prefer `pull_request` (read-only token, no secrets) over `pull_request_target` for *anything* that touches PR content. Reserve `pull_request_target` for metadata-only operations (labeling, commenting based on path globs, dependency-review on the diff metadata).
- **Never check out the PR head SHA** in a job that has secrets in its environment.
- Keep `on.roles:` at its default — do not set `on.roles: all` to "be friendly to the community"; pair restrictive roles with a separate, narrower workflow if community-facing commands are needed.
- Pin `safe-outputs:` to the minimum required and audit the resulting blast radius via §6.4.
- For any workflow that *must* be powerful, gate the agent step on `github.event.pull_request.head.repo.fork == false` so it refuses to act on cross-fork PRs entirely — and rely on the approval gate **only as defense-in-depth**, never as the primary control.

---

## 4. Operating Within a Fork ⚠️

When you fork a repository, a copy of every workflow file (`.github/workflows/**`) comes with the fork — including agentic workflows. **GitHub then treats your fork as its own first-class repository.** Any event you trigger *within your fork* fires the workflow inside *your fork*, with *your fork's* secrets and *your fork's* `GITHUB_TOKEN`. This is structurally separate from the cross-fork PR scenario in §3 and is frequently a surprise.

### 4.1 What fires when you operate inside your own fork

| Activity inside your fork | Workflow fires? | Runs as | Token / secrets |
|---|---|---|---|
| You push commits to a branch in your fork | ✅ on `push` | You (fork owner) | Your fork's secrets, full write `GITHUB_TOKEN` to your fork |
| You open a PR `feature → main` **within your fork** (self-review pattern) | ✅ on `pull_request` (this is **not** treated as a cross-fork PR — both refs are in your fork) | You | Your fork's secrets, full write token |
| You open an issue, comment, react in your fork | ✅ on `issues`, `issue_comment`, etc. | You | Your fork's secrets, full write token |
| You apply a slash command in your own fork | ✅ — and **`on.roles:` defaults still apply**, but you're admin of your own fork, so the membership check passes | You | Your fork's secrets, full write token |
| Scheduled (cron) workflows in your fork | ❌ by default — GitHub disables `schedule` triggers on forks until you re-enable them in the Actions tab | n/a | n/a |
| You open a PR from your fork **back to upstream** | ✅ on upstream's `pull_request` (with read-only token, no upstream secrets); ✅ on upstream's `pull_request_target` 🛑 (with full upstream secrets) | Upstream | Upstream's secrets (with the cross-fork caveats from §7.4 / §7.5) |

**Key consequence:** Workflows you forked from upstream — agentic or otherwise — will **start running for you on routine activity in your fork**, often unexpectedly. They run as *you* (or whatever PAT pool *you* configured), not as the upstream owner. This is harmless when secrets are unset (most workflows fail fast), but is an unwanted surprise when:

- The forked workflow contains a `bash` step that pushes to a remote, posts to Slack, files a GitHub issue, or sends an email.
- The forked workflow uses GitHub Apps that you happened to install on your fork.
- You're using your fork as a sandbox to "study" the upstream workflows and they fire while you read.
- You reuse the upstream's PAT-pool secret names and have analogous secrets in your fork — your secrets get used by code you didn't author.

### 4.2 The `if: workflow_dispatch || not-a-fork` guard pattern

The simplest and most reliable defense is a top-level job condition that prevents every workflow from running in any fork unless explicitly invoked manually:

```yaml
on:
  pull_request:
  push:
  issue_comment:
  workflow_dispatch:

jobs:
  guard:
    if: ${{ github.event_name == 'workflow_dispatch' || github.event.repository.fork == false }}
    # ...rest of workflow
```

Equivalent and arguably stricter — pin the workflow to the **specific upstream `owner/repo`**:

```yaml
if: ${{ github.event_name == 'workflow_dispatch' || github.repository == 'OWNER/REPO' }}
```

**Why both clauses?**

- `github.event.repository.fork == false` (or `github.repository == 'OWNER/REPO'`) prevents the workflow from running on routine events inside any fork.
- `github.event_name == 'workflow_dispatch'` is the explicit escape hatch: if a forker *intends* to run the workflow themselves to test their changes, they invoke it manually from the Actions tab. They get the consequences of their own action, not a surprise.

**For agentic workflows specifically:** add the same `if:` to the `pre-activation` job in the auto-injected `on.steps:` block, so even the activation runner doesn't spin up in forks. Without this, the workflow still consumes a runner minute in every fork on every event — and if the fork owner happens to have analogously-named secrets, the agent step itself may run.

> 💡 **Idiomatic placement.** Apply the guard at the *job* level (or on the activation/pre-activation job for gh-aw), **not** the step level. A job-level `if:` skips the entire job (no runner spun up); a step-level `if:` still consumes a runner.

### 4.3 What the fork owner *cannot* do to you (the upstream)

Forking does not give the fork owner any new privileges over the upstream. They still have to come back through the §3 cross-fork PR gate to affect upstream. The risks in §4 are entirely **about the fork owner being surprised by their own forked copies of your workflows**, and (transitively) about the *upstream* being judged poorly for shipping workflows that misbehave in forks.

---

## 5. Concurrency and Race Conditions

GitHub's concurrency model[^concurrency]:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.event.issue.number }}
  cancel-in-progress: true
```

- **One run per group.** Newer queued runs cancel older queued runs in the same group.
- With `cancel-in-progress: true`, a newer run also cancels an in-progress older run.
- **Cancellation is asynchronous**: GitHub sends `SIGTERM`, waits up to 7500 ms, then `SIGKILL`. Already-running steps may complete; in-flight API calls may succeed.

### 5.1 The "non-matching cancels matching" race

Concrete scenario: workflow `triage.md` has `slash_command: triage` and `concurrency.group: triage-${{ github.event.issue.number }}` with `cancel-in-progress: true`.

```
T+0  s: User comments "/triage" on issue #42 → run A starts; pre-activation matches; agent job begins.
T+10 s: Same user adds a clarifying comment "Actually, I meant the 1.x version." (no slash command).
        → issue_comment event fires; run B is queued.
        → Concurrency group "triage-42" already has run A in-progress → A is canceled.
        → Run B's pre-activation runs, sees no slash command, exits skip.
        → Net result: the triage agent for /triage was killed mid-run by an unrelated comment.
```

This is the **non-matching-cancels-matching pathology** the user described. Mitigations:

1. **Narrow `slash_command` `events:`** so the workflow doesn't subscribe to events that can't possibly match — e.g., `events: [issue_comment]` when the slash command will only ever appear in comments.
2. **Make the concurrency group depend on the activation outcome.** This is hard — `concurrency:` is evaluated *before* the job starts and cannot reference job outputs. The practical workaround is to make the group narrower (e.g., include `${{ github.event.comment.id }}`) so unrelated events don't share the group.
3. **Don't set `cancel-in-progress: true`** for slash-command workflows; let them queue.
4. **Use `lock-for-agent: true`** for issue/PR mutations so even concurrent runs serialize against the issue lock.
5. **Make agent-side mutations idempotent** (use deterministic comment markers, upsert labels, check for existing PRs) so a re-run produces the same end state.

### 5.2 The pre-cancellation race

Even without conflicting groups, **steps that have already started will complete** before cancellation lands. This means:

- An agent that has already posted a comment cannot un-post it.
- A `safe-outputs.create-pull-request` that has already run cannot un-create the PR.
- A `bash` step mid-execution may write partial files, push partial commits, etc.

Concurrency is not a substitute for idempotency.

---

## 6. Authorization, Roles, and Read-Only Contributors

The user's emphasis on **read-only contributors** is critical, and the answer has two layers: **what GitHub itself permits** and **what gh-aw layers on top by default**.

### 6.1 What each role can do *that fires a workflow* (raw GitHub permissions)

Here is what each role can do *that fires a workflow* — at the GitHub event layer, before any gh-aw filtering:

| Action | Read | Triage | Write | Maintain | Admin |
|---|---|---|---|---|---|
| Open an issue | ✅ | ✅ | ✅ | ✅ | ✅ |
| Edit own issue body/title | ✅ | ✅ | ✅ | ✅ | ✅ |
| Close own issue | ✅ | ✅ | ✅ | ✅ | ✅ |
| Comment on issue/PR/discussion | ✅ | ✅ | ✅ | ✅ | ✅ |
| Edit own comment | ✅ | ✅ | ✅ | ✅ | ✅ |
| React with emoji | ✅ | ✅ | ✅ | ✅ | ✅ |
| Open a PR (from fork or, if collaborator, from branch) | ✅ (via fork) | ✅ | ✅ | ✅ | ✅ |
| Edit own PR body | ✅ | ✅ | ✅ | ✅ | ✅ |
| `/slash-command` in any comment/body they author | ✅ | ✅ | ✅ | ✅ | ✅ |
| Apply a label | ❌ | ✅ | ✅ | ✅ | ✅ |
| Approve/dismiss reviews | ❌ | ❌ | ✅ | ✅ | ✅ |
| Push to repo branches | ❌ | ❌ | ✅ | ✅ | ✅ |
| Invoke `workflow_dispatch` | ❌ | ❌ | ✅ | ✅ | ✅ |
| Click "Approve and run workflows" | ❌ | ❌ | ✅ | ✅ | ✅ |

**Key observation:** A reader can fire any `slash_command:` and any `issues`/`issue_comment`/`pull_request`/`discussion_comment`/`reaction`-driven workflow, just by typing or clicking. They cannot fire `label_command:` or `workflow_dispatch:` triggered ones.

**Key observation (raw layer):** A reader can fire any `slash_command:` and any `issues`/`issue_comment`/`pull_request`/`discussion_comment`/`reaction`-driven workflow, just by typing or clicking. They cannot fire `label_command:` or `workflow_dispatch:` triggered ones.

### 6.2 gh-aw's `on.roles:` allowlist (the *primary* authz mechanism)

gh-aw automatically injects an **activation-job membership check** into any workflow whose triggers include "unsafe" events (issues, comments, PRs, discussions, slash/label commands). The allowlist is configured by `on.roles:` and **defaults to `[admin, maintainer, write]`**[^role-checks-go]. Behaviors:

- **String form** — `on.roles: write` (single role) or `on.roles: all` (special value disabling the check entirely).
- **Array form** — `on.roles: [admin, maintainer, write]` (the default) or `on.roles: [admin, maintainer, write, triage, read]` to broaden.
- **Inverse field** — `on.skip-roles: [role,...]` blocks specific roles from a wider allowlist (e.g., `on.roles: all` + `on.skip-roles: [read]`).
- **The check still consumes a runner.** Like other activation-job filters, the workflow *runs* and the activation job calls the GitHub repo permission API; the agent job is then skipped if the actor's role is not in the allowlist.
- **`on.roles: all` is required for any chat-style workflow that intentionally serves read-only contributors** — for example, a self-service `/help` command or a community FAQ bot.
- **Does not protect against `workflow_run` chained workflows** (which run as the base repo, not as the original actor).

### 6.3 GitHub permission roles (vocabulary for this section)

| Role | Default in `on.roles`? | Notes |
|---|---|---|
| `admin` | ✅ | Full control |
| `maintainer` (a.k.a. `maintain`) | ✅ | Can manage repo without billing/admin |
| `write` | ✅ | Can push, label, approve PRs, dispatch workflows |
| `triage` | ❌ | Can manage issues/PRs (label, assign, close) but cannot push |
| `read` | ❌ | Can open issues/PRs/comments/reactions only |
| (anonymous fork contributor) | ❌ | Treated as `read` for membership API purposes |
| `bot` actor | (inherits the bot's permission level) | Use `on.skip-bots:` to block specific bot logins |

**`triage` is invisible in the default allowlist** — that means a workflow like `label_command:` (which requires triage to apply the label in the first place) will *fire* but its activation job will *deny* a triage user unless `on.roles:` is broadened. This is a frequent footgun.

### 6.4 Consolidated read-only / fork contributor write surface 🛑

The single most important security insight in this guide:

> **What the agent can do is determined by the workflow's `permissions:` and `safe-outputs:` declarations — NOT by the actor who fired it.** When a workflow accepts a read-only or fork contributor as the trigger (i.e., `on.roles: all`, no actor check, or no fork guard), that contributor effectively gets **bot-level write access** to anything the workflow grants the agent.

This is why the gh-aw `on.roles:` default of `[admin, maintainer, write]` exists — it's a *deny-by-default* gate that prevents a read user from inducing the bot to act with the bot's elevated permissions. The moment you set `on.roles: all` (or write a workflow that ignores actor identity), every mutation in the table below becomes reachable by anyone who can fire the trigger.

**Capability matrix.** Assuming the workflow grants the relevant `permissions:` and exposes the relevant safe-output, can a **read-only contributor** (or anonymous fork contributor) cause the listed action by firing each gh-aw trigger?

Legend: ✅ = reachable; ⚠️ = reachable but with caveats; ❌ = not reachable via this trigger; 🛑 = reachable AND the workflow runs with **upstream secrets** (the high-impact pwn vector).

| Action the agent can take | `slash_command` on issue/comment | `slash_command` on own PR (cross-fork) | `issues` (opened/edited) | `issue_comment` (PR comment, cross-fork) | `discussion` / `discussion_comment` | `pull_request` (cross-fork) | `pull_request_target` (cross-fork) |
|---|---|---|---|---|---|---|---|
| Post a comment on the triggering item | ✅ | ✅ | ✅ | 🛑 | ✅ | ⚠️ read-only token | 🛑 |
| Add or remove labels (`add-labels`) | ✅ | ✅ | ✅ | 🛑 | ✅ | ⚠️ | 🛑 |
| Edit issue/PR title or body (`update-issue`) | ✅ | ✅ | ✅ | 🛑 | ❌ (discussions don't have an analog safe-output) | ⚠️ | 🛑 |
| Close / reopen issue or PR (`update-issue`) | ✅ | ✅ | ✅ | 🛑 | ❌ | ⚠️ | 🛑 |
| Edit or delete *existing* comments | ❌ no safe-output for editing/deleting comments | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Create a new issue (`create-issue`) | ✅ | ✅ | ✅ | 🛑 | ✅ | ⚠️ | 🛑 |
| Create a new pull request (`create-pull-request`) | ✅ | ✅ | ✅ | 🛑 | ✅ | ⚠️ | 🛑 |
| Push commits to the PR branch (`push-to-pull-request-branch`) | ❌ (no PR context) | ⚠️ only when PR is from same repo, not cross-fork | ❌ | ⚠️ same-repo only | ❌ | ❌ cross-fork is blocked | ⚠️ same-repo only |
| Edit code on the PR (commit changes) | ❌ | ⚠️ same caveat as push | ❌ | ⚠️ same | ❌ | ❌ | ⚠️ same |
| Push to the upstream default branch | ❌ unless workflow runs `git push` from a `bash` step (very ill-advised) | same | same | same | same | same | same |
| Create a discussion (`create-discussion`) | ✅ | ✅ | ✅ | 🛑 | ✅ | ⚠️ | 🛑 |
| Trigger a downstream `workflow_dispatch` (via `gh workflow run`) | ✅ if `permissions: actions: write` | same | same | 🛑 | same | ⚠️ | 🛑 |
| Read repository secrets (and exfiltrate via comment) | ✅ — secrets are in the agent's `env`, can be echoed into any output | same | same | 🛑 | same | ❌ no secrets on fork PRs | 🛑 |
| Approve a pull request | ❌ — a workflow's `GITHUB_TOKEN` cannot approve PRs (GitHub policy) | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Merge a pull request | ✅ if `permissions: pull-requests: write, contents: write` and PR is mergeable | same | same | 🛑 | n/a | ⚠️ | 🛑 |
| Add/remove repo collaborators, change branch protection | ❌ — `GITHUB_TOKEN` lacks admin scopes; requires a separate PAT | same | same | same | same | same | same |

**How to read the 🛑 column intersections.** A 🛑 means: a contributor with **only read access** (or none — anonymous fork contributors via PR comments) can, just by typing a `/command` or comment, induce the bot to perform the listed mutation **using the upstream's secrets and write token**. This is the gh-aw-flavored re-statement of the classic "pwn requests" class[^pwn-requests] and is the entire reason `on.roles:` defaults to `[admin, maintainer, write]`.

**Defenses, in priority order:**

1. **Leave `on.roles:` at its default.** Don't set `on.roles: all` unless you've thought through every cell of the matrix above for your workflow's actual `permissions:` and safe-outputs.
2. **Minimize `permissions:`** to the smallest set the agent actually needs.
3. **Minimize `safe-outputs:`** to only the structured mutations the workflow needs to make.
4. **For PR-touching workflows: never check out the PR head SHA in the same job that has secrets.** Use `pull_request` (read-only token) for code analysis, and a separate `pull_request_target` job *without* checkout for the comment/label posting.
5. **Add an explicit fork guard** (`if: github.event.pull_request.head.repo.fork == false`) for any agent step that should refuse to act on cross-fork PRs.

---

## 7. Trigger-by-Trigger Risk Profile

The dimensions the user cares about: **security/authz**, **approve-and-run gate**, **when it runs (runner & token cost)**, **concurrency/race**, **unintended consequences**, **forks**, **mixing with `workflow_dispatch`**.

### 7.1 `workflow_dispatch`
- **Authz:** Write access required. Cannot be invoked by readers/forks.
- **Approve gate:** N/A.
- **Runs:** Only when explicitly invoked or when paired with another trigger that auto-injects it (most gh-aw shorthands do).
- **Concurrency:** Standard. Manually-fired runs respect groups.
- **Forks:** Cannot dispatch on a fork's branch from upstream; the fork owner can dispatch in their own fork.
- **Unintended:** Inputs are user-controlled strings → never pass them un-quoted into shell. Use `${{ github.event.inputs.x }}` only inside `env:` blocks.

### 7.2 `schedule`
- **Authz:** None — runs as the workflow file on the default branch.
- **Approve gate:** N/A.
- **Runs:** On cron; **disabled after 60 days of repo inactivity**[^events-docs].
- **Concurrency:** Skewed in gh-aw via fuzzy schedules to avoid thundering herds.
- **Forks:** Schedules in forks are disabled by default.
- **Unintended:** Long-running schedules + `stop-after:` mismatch can leave a workflow active longer than intended after a recompile.

### 7.3 `push`
- **Authz:** Push access required.
- **Approve gate:** N/A.
- **Runs:** On every matching push; can be filtered by `branches`, `tags`, `paths`.
- **Concurrency:** Use `${{ github.ref }}` group + `cancel-in-progress: true` for CI.
- **Forks:** Push events on forks fire on the fork; do not propagate.

### 7.4 `pull_request`
- **Authz:** Anyone who can open a PR (incl. fork contributors).
- **Approve gate:** YES for first-time / outside contributors per org policy[^approval-policy].
- **Runs:** On the **PR's merge ref** with **read-only `GITHUB_TOKEN`** and **no secrets** for fork PRs[^token-permissions].
- **Concurrency:** `${{ github.workflow }}-${{ github.event.pull_request.number }}` with cancel-in-progress is the standard CI pattern.
- **Forks:** Allowed; this is the *safe* PR trigger.
- **Unintended:** None major — read-only token + no secrets is the safety net.

### 7.5 `pull_request_target` 🛑 ⚠️ ⚠️ ⚠️
- **Authz:** Same actors as `pull_request`.
- **Approve gate:** Same as `pull_request`.
- **Runs:** On the **base ref** with **full `GITHUB_TOKEN` and access to all secrets**.
- **Concurrency:** Same patterns.
- **Forks:** Allowed — but you must **never check out the PR's head SHA** or run any code from it[^pwn-requests]. Safe uses: labeling, commenting, dependency review on the diff metadata.
- **Unintended:** 🛑 Most-exploited GitHub Actions vulnerability class — "pwn requests." Combined with §6.4: a fork contributor with **no repo access at all** can induce the workflow to execute arbitrary writes (comment, label, push to base, merge a different PR, etc.) using upstream secrets. **Always pair with `on.roles:` (default) and a fork-head guard.**

### 7.6 `issues`
- **Authz:** Read access (anyone can open issues; only own issues can be closed/edited as a reader).
- **Approve gate:** N/A.
- **Runs:** Always.
- **Concurrency:** Group on `${{ github.event.issue.number }}` to serialize per-issue work.
- **Forks:** N/A (issues live on the upstream).
- **Unintended:** `issue.body`/`issue.title` are user-controlled — use gh-aw's `steps.sanitized.outputs.text` rather than raw `${{ github.event.issue.body }}` to avoid prompt-injection / template-injection[^command-doc].

### 7.7 `issue_comment` 🛑 ⚠️
- **Authz:** Read.
- **Approve gate:** N/A — even on forks.
- **Runs:** On every comment on issues *and* PRs; distinguish via `github.event.issue.pull_request != null`.
- **Concurrency:** Group on `${{ github.event.issue.number }}` — but be aware of §5.1.
- **Forks:** Comments on PRs from forks fire on the upstream and run with **upstream's secrets/token**.
- **Unintended:** 🛑 A PR-author who is a fork contributor (with **no** repo access) can comment on their own PR and trigger `issue_comment` workflows that run with upstream secrets. This is the second pwn-requests vector. See §6.4 for the full capability matrix. Defenses (in order): keep `on.roles:` at default (excludes anonymous fork actors), add `skip-roles:` for fine-grained blocks, never check out PR head SHA in the same job, audit `safe-outputs:` for the actual blast radius.

### 7.8 `pull_request_review` / `pull_request_review_comment` 🛑 ⚠️
- Same caveats as `issue_comment` (§7.7). Reviews fire even from fork contributors with upstream secrets — same 🛑 pwn vector applies.

### 7.9 `discussion` / `discussion_comment`
- **Authz:** Read.
- **Approve gate:** N/A.
- **Runs:** On the upstream repo only.
- **Forks:** N/A.
- **Unintended:** Same content-sanitization concerns; lower stakes because typically no PR/code context.

### 7.10 `workflow_run` 🛑 ⚠️
- **Authz:** Indirect (whoever can trigger the upstream workflow).
- **Approve gate:** N/A.
- **Runs:** With **base repo's secrets and write token**, on the **default branch** of the workflow file.
- **Concurrency:** Standard.
- **Forks:** Fires for runs originating from fork PRs — this is the canonical "smuggle untrusted artifacts back into a trusted context" trigger[^pwn-requests].
- **Unintended:** Downloading and operating on artifacts from the triggering run is the classic vulnerability. gh-aw enforces fork/repository-ID checks and requires `branches:` to be set[^triggers-doc].

### 7.11 `repository_dispatch`
- **Authz:** PAT with `repo` scope.
- **Approve gate:** N/A.
- **Runs:** With base secrets.
- **Concurrency:** Standard.
- **Forks:** N/A.
- **Unintended:** Anyone with the PAT can fire arbitrary `event_type` strings.

### 7.12 `workflow_call`
- **Authz:** Caller's authz.
- Effectively a function call between workflows — security inherits caller.

### 7.13 `slash_command:` (gh-aw) 🛑 ⚠️
- **Authz:** Inherits the underlying event (read for comments/issues/discussions, fork rules for PRs). **gh-aw auto-injects `on.roles: [admin, maintainer, write]`** — read/triage actors fire the workflow but the agent step is skipped unless `on.roles:` is broadened or set to `all`. **Setting `on.roles: all` opens every cell of the §6.4 capability matrix to anyone who can post a comment.**
- **Approve gate:** Inherits — comment-only commands escape the gate; PR-body commands hit it.
- **Runs:** On *every* event in the (default-or-narrowed) `events:` list. Activation job filters.
- **Concurrency:** See §5.1 — broad event subscription + shared concurrency group is the pathology.
- **Forks:** Same as comments/PRs — comment-on-own-fork-PR is the dangerous path 🛑.
- **Unintended:** Aliased commands (`name: [a, b, c]`) multiply the surface; first-word matching means `\`/deploy\`` (in markdown code) does not trigger but `/deploy now` does.
- **Unintended:** Aliased commands (`name: [a, b, c]`) multiply the surface; first-word matching means `\`/deploy\`` (in markdown code) does not trigger but `/deploy now` does.

### 7.14 `label_command:` (gh-aw)
- **Authz:** Triage+ (label application requires triage role).
- **Approve gate:** N/A.
- **Runs:** When a matching label is applied; auto-removes label so the command is one-shot.
- **Concurrency:** Group on `${{ github.event.issue.number }}`.
- **Forks:** N/A — labels are upstream.
- **Unintended:** `remove_label: false` turns it into a state marker rather than a command — different semantics. Triage role is the de-facto authz boundary.

---

## 7. Demonstration Corpus for This Repo

To turn this into the "build a workflow per scenario" effort the user described, organize by **(trigger × actor × outcome)** triples. Suggested SQLite schema for tracking, and a directory layout under `.github/workflows/triggers-demo/`:

```
triggers-demo/
  std-push.md
  std-pull_request-fork.md
  std-pull_request_target-fork-safe.md
  std-issues-opened.md
  std-issue_comment.md
  std-issue_comment-pr.md
  std-pr-review.md
  std-pr-review-comment.md
  std-discussion-created.md
  std-discussion-comment.md
  std-schedule-fuzzy.md
  std-workflow_dispatch.md
  std-workflow_run.md
  std-repository_dispatch.md
  std-release.md
  std-fork.md
  std-watch.md
  ...
  ghaw-slash-broad-default.md         # Demonstrates skipped-run noise
  ghaw-slash-narrowed-events.md       # events: [issue_comment]
  ghaw-slash-on-roles-default.md      # Demonstrates default [admin,maintainer,write] gate
  ghaw-slash-on-roles-all.md          # roles: all (intentionally serves read users)
  ghaw-slash-on-roles-with-triage.md  # Adds triage to the allowlist
  ghaw-slash-with-skip-roles.md       # on.roles: all + skip-roles: [read]
  ghaw-slash-on-pr-body.md            # Triggers approve-and-run gate from forks
  ghaw-slash-multi-name.md            # name: [a,b,c]
  ghaw-label-command-deploy.md
  ghaw-label-command-state-marker.md  # remove_label: false
  ghaw-manual-approval.md
  ghaw-skip-if-match.md
  ghaw-stop-after.md
  ghaw-status-comment.md
  ghaw-reaction-only.md
  ghaw-lock-for-agent.md
  ghaw-forks-trusted-org.md
  ghaw-on-steps-prefilter.md
  race-concurrency-cancel.md          # Two workflows demonstrating §5.1
  race-idempotent-mutations.md        # Counter-example with safe-outputs deduping
```

### Suggested tracking dimensions per workflow

For each scenario file, record:

1. **Trigger spec** (the `on:` block)
2. **Actor classes that can fire it** (anonymous via fork, read, triage, write, maintain, admin, bot)
3. **Approve-and-run gate?** (yes/no, when)
4. **Activation job runtime** (skipped vs run distribution)
5. **Concurrency group recommendation**
6. **Race vulnerabilities**
7. **Token/secrets exposure** (read-only fork vs full base)
8. **Mixing-with-`workflow_dispatch` notes**
9. **Observed user findings** (open question — to be filled by the interview)

This is a natural fit for the `todos` SQL table or a custom `trigger_scenarios` table in this session.

---

## Footnotes

[^triggers-doc]: gh-aw **Triggers** reference — `https://raw.githubusercontent.com/githubnext/gh-aw/main/docs/src/content/docs/reference/triggers.md` (fetched 2026-04-17). Sections cited: *Trigger Types*, *Command Triggers (`slash_command:`)*, *Label Command Trigger (`label_command:`)*, *Reactions*, *Status Comments*, *Stop After Configuration*, *Manual Approval Gates*, *Skip-If-Match*, *Skip-If-No-Match*, *Pre-Activation Steps*, *Trigger Shorthands*.

[^command-doc]: gh-aw **Command Triggers** reference — `https://raw.githubusercontent.com/githubnext/gh-aw/main/docs/src/content/docs/reference/command-triggers.md` (fetched 2026-04-17). Quoted: *"By default, command triggers listen to all comment-related events, which can create skipped runs in the Actions UI"*; *"The command must be the first word of the comment or body text to trigger the workflow"*; supported events list (`issues`, `issue_comment`, `pull_request`, `pull_request_comment`, `pull_request_review_comment`, `discussion`, `discussion_comment`, `*`).

[^events-docs]: GitHub Docs, **Events that trigger workflows** — `https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows` (fetched 2026-04-17). Source for the standard event table including activity types, "default branch only" flag, and the schedule-disabled-after-60-days rule.

[^frontmatter-doc]: gh-aw **Frontmatter** reference — `https://raw.githubusercontent.com/github/gh-aw/main/docs/src/content/docs/reference/frontmatter.md` (fetched 2026-04-17). Lists `skip-roles`, `skip-bots`, `skip-if-match`, `skip-if-no-match`, `manual-approval`, `forks`, `stop-after`, `reaction`, `status-comment`, `on.steps`, `on.permissions`, `on.github-token`, `on.github-app`. (Note: the rendered frontmatter doc does not currently list `on.roles:`; that field is documented authoritatively in the source code — see [^role-checks-go].)

[^role-checks-go]: gh-aw source — [github/gh-aw/`pkg/workflow/role_checks.go`](https://github.com/github/gh-aw/blob/main/pkg/workflow/role_checks.go). Authoritative for the `on.roles:` semantics: default allowlist `["admin", "maintainer", "write"]` (line ~109), special string value `"all"` disables the check (line ~118), per-event applicability via `needsRoleCheck` (line ~300), `hasSafeEventsOnly` short-circuit for safe-only triggers (line ~325). Also documents the parallel `ignored-roles:` field (default `["admin", "maintain", "write"]`) used for activation rate-limit exemption.

[^approval-docs]: GitHub Docs, **Approving workflow runs from public forks** — `https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-workflow-runs/approving-workflow-runs-from-public-forks`. Describes the *Approve and run workflows* button and the conditions under which it appears.

[^approval-policy]: GitHub Docs, **Disabling or limiting GitHub Actions for your repository / organization → Approving workflow runs from public forks** — `https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository`. Source of the four approval-policy levels (first-time-contributor-new-to-GitHub, first-time-contributor, all outside collaborators, all external contributors).

[^token-permissions]: GitHub Docs, **Automatic token authentication → Permissions for the GITHUB_TOKEN** — `https://docs.github.com/en/actions/security-guides/automatic-token-authentication`. Source for the read-only-and-no-secrets behavior on fork PRs under `pull_request`.

[^pwn-requests]: GitHub Security Lab, **Keeping your GitHub Actions and workflows secure: Preventing pwn requests** — `https://securitylab.github.com/research/github-actions-preventing-pwn-requests/`. Authoritative analysis of `pull_request_target`, `workflow_run`, and `issue_comment`-from-fork attack patterns.

[^concurrency]: GitHub Docs, **Using concurrency, expressions, and a test matrix → Using concurrency** — `https://docs.github.com/en/actions/using-jobs/using-concurrency`. Describes group-keyed serialization and `cancel-in-progress` semantics; cancellation is asynchronous (SIGTERM → 7500 ms grace → SIGKILL).
