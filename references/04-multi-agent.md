# Multi-Agent Orchestration Patterns

---

## The Core Insight

"Multi-agent" is not a single pattern. It is a family of distinct topologies, each with different semantics around context sharing, lifecycle, cost, and trust:

| Topology | Parent context shared? | Cache shared? | Lifecycle | Typical use |
|---|---|---|---|---|
| **Fork** | Yes (full history) | Yes (byte-identical prefix) | Inline or background | Parallel scaling the same conversation |
| **Fresh subagent** | No | Partially | Inline or background | Specialized task with different system prompt |
| **Worker (coordinator pattern)** | No (only the directive) | No | Always background | Decomposed work under a non-executing parent |
| **Team** | Shared message bus | No | Long-running, persistent | Multiple agents collaborating on a shared problem |
| **Remote** | No (RPC boundary) | No | Cross-machine async | Offload to a server, integrate via Slack/PR bots |
| **Scheduled** | No (cold start) | No | Recurring, time-driven | Babysitting, monitoring, drip-fed automation |

A production multi-agent system uses several of these, not one. The decision is per-task, not per-system: "this exploration is a fork; this verification is a fresh subagent; this 12-hour migration runs in a remote worker."

---

## Pattern 1: Agent Definition as a Schema

Every agent — built-in, user-defined, or synthetic — is described by a schema, not subclassed:

```typescript
type AgentDefinition = {
  agentType: string              // Unique identifier
  whenToUse: string              // Description for the orchestrator model
  tools: string[]                // Allowlist (or ['*'] for parent's exact pool)
  useExactTools?: boolean        // If true, inherit parent's tool pool byte-identical
  disallowedTools?: string[]     // Denylist applied after allowlist
  permissionMode: PermissionMode // 'default' | 'bubble' | 'plan' | ...
  isolation?: 'none' | 'worktree'// Filesystem isolation
  model: string                  // 'inherit' or specific model
  maxTurns: number               // Runaway prevention
  systemPrompt: string | () => string  // Static or computed
  background?: boolean           // Default execution mode
  source: 'built-in' | 'user' | 'project' | 'mcp'
}
```

**The `whenToUse` field is the most important.** It is what the orchestrator model reads when selecting which agent type to spawn — essentially the "tool description" for agents-as-tools. Vague `whenToUse` strings produce confused orchestrators that pick the wrong agent.

**`tools: ['*']` with `useExactTools: true`** is the fork pattern — the child gets the parent's exact tool pool, byte-for-byte. This is required for prompt cache sharing (see Pattern 2). Any other configuration produces a different tool list, which produces a different prefix hash, which busts the cache.

---

## Pattern 2: Fork — Byte-Identical Prefix Sharing

**The fork mechanism is not "child inherits parent's prompt."** It is "child sends a byte-identical prefix to the API so the cache is shared."

Three implementation details make this work:

**1. Thread the rendered prompt, don't recompute it.** When the fork spawns, the parent's already-rendered system prompt bytes travel to the child via `toolUseContext.renderedSystemPrompt`. The child does NOT call its own `getSystemPrompt()` — that call could diverge for many reasons (feature flag cold→warm transition, time-dependent section, mid-session settings change), and any divergence costs the entire prefix cache.

**2. Build conversation history with identical placeholders.** The fork constructs the child's first request as:

```
[ ...parent_history ]
assistant: <all parent's tool_use blocks, thinking, text>
user:
  tool_result(tool_use_id=A, content="Fork started — processing in background")
  tool_result(tool_use_id=B, content="Fork started — processing in background")
  ...
  text: "<fork-boilerplate>STOP. READ THIS FIRST. You are a forked worker..."
  text: "<fork-directive>Your specific task: ..."
```

Every fork child gets identical placeholder tool_results — the only difference between children is the final directive text. This pushes the cache hit boundary all the way to the directive, maximizing shared bytes.

**3. Recursion is blocked structurally.** Fork children keep the spawning tool in their pool (for cache identity), so the framework has to prevent recursive forking somewhere else. The standard mechanism is detecting the boilerplate tag in the child's history: if `<fork-boilerplate>` appears, the child is a fork, and fork attempts within fork are rejected.

**The fork boilerplate prompt is a discipline document.** Forks are instructed to:

- Execute directly, not spawn more sub-agents
- Not converse, not ask questions, not editorialize
- Use tools silently between the directive and the final report
- Commit any file changes before reporting, include the commit hash
- Stay strictly within the assigned scope
- Begin the response with "Scope:" — a structural anchor for parsing
- Limit reports to ~500 words

The output format is a labelled flat list (Scope, Result, Key files, Files changed, Issues), not free prose. This makes worker output programmatically parseable.

---

## Pattern 3: Fresh Subagent — Different Persona, Different Tools

When the task is not "more of the same work in parallel" but "this specific kind of task with a different mindset," fork is the wrong tool. The right tool is a fresh subagent.

A fresh subagent has its own `agentType`, its own system prompt, its own (possibly smaller) tool pool. It does not inherit the parent's conversation history (or inherits only a curated subset). It does not share the prompt cache.

**Examples of agents that should be fresh, not forked:**

- **Verifier agent.** Runs adversarial tests against the parent's work. Must not see the parent's reasoning — that would bias the verification.
- **Reviewer agent.** Reviews code with a clean perspective. Inheriting the parent's "I just wrote this" context defeats the purpose.
- **Planning agent.** Builds an implementation plan in isolation, without the parent's existing assumptions.
- **Doc-writer agent.** Different writing style, different audience, different tool pool (no shell, no edits).

Fresh subagents are more expensive than forks (no cache sharing) but cheaper than they look — they typically have much shorter contexts, so the savings on history compensate for the prefix cache miss.

**Use the orchestrator's whenToUse text to teach selection.** A typical `whenToUse` for a verifier reads:

> "Fast read-only verification agent. Spawn this AFTER non-trivial implementation work to confirm it actually does what it's supposed to. The verifier runs tests, checks file existence, and issues a PASS/FAIL/PARTIAL verdict. You cannot self-verify your own work."

Specific, with WHEN and a structural constraint (cannot self-verify).

---

## Pattern 4: Worker Topology — Coordinator Never Executes

A different paradigm: the parent never executes anything. The parent is a **coordinator**. It reads requirements, decomposes them, dispatches workers, synthesizes their results, and reports to the user. It does not call file tools, shell tools, or web tools directly.

The coordinator's tool pool is small and orchestration-shaped:

- A spawning tool to launch a worker
- A send-message tool to continue an existing worker
- A stop tool to kill a worker
- Optional: subscription tools (PR events, build status)

The workers do the actual work. Each worker is its own fresh subagent with full execution tools.

**Why this topology?** Three reasons:

1. **Parallelism scales linearly.** The coordinator launches three workers in one turn; they run concurrently; the coordinator gets three results. A single-agent execution serializes the same work.

2. **The coordinator's context stays small.** Worker contexts accumulate the noise of file reads, command output, retries. The coordinator only sees worker *summaries*. A coordinator running an 8-hour migration uses less context than a single agent doing 30 minutes of the same work.

3. **Worker failures are contained.** A worker that derails or hits an infinite loop does not corrupt the coordinator's context. The coordinator sees "worker X failed: timeout" and can decide to retry, skip, or escalate.

**The interaction protocol.** Worker results arrive as `user`-role messages with a `<task-notification>` XML envelope:

```xml
<task-notification>
  <task-id>agent-a1b</task-id>
  <status>completed|failed|killed</status>
  <summary>{human-readable status}</summary>
  <result>{worker's final text}</result>
  <usage>
    <total_tokens>N</total_tokens>
    <tool_uses>N</tool_uses>
    <duration_ms>N</duration_ms>
  </usage>
</task-notification>
```

The coordinator's system prompt teaches it to distinguish these from real user messages by the opening tag. The `<task-id>` lets the coordinator continue a worker via `SendMessage({ to: agent-a1b, message: "..." })`, reusing the worker's loaded context for follow-up work.

**The coordinator's four-phase workflow.** This pattern works best when the work is decomposable into:

| Phase | Who | Purpose |
|---|---|---|
| **Research** | Workers (parallel) | Investigate, find files, understand the problem |
| **Synthesis** | **Coordinator** | Read findings, design the approach, write specs |
| **Implementation** | Workers (parallel or serialized) | Make targeted changes per spec |
| **Verification** | Workers | Test that changes work |

Synthesis is the *only* phase where the coordinator does cognitive work directly. It is the phase that cannot be delegated because it requires integrating findings across workers.

---

## Pattern 5: Permission Modes for Sub-Agents

A sub-agent can run in different permission modes from its parent:

| Mode | Behavior | Use case |
|---|---|---|
| `inherit` | Same as parent | Default for forks |
| `default` | Standard ask-for-writes | Fresh subagents in interactive contexts |
| `bubble` | Permission prompts surface to parent's terminal | Forks of an interactive session |
| `plan` | Read-only execution | Research-only workers |
| `bypassPermissions` | Auto-approve everything | Trusted automation, must be operator-enabled |

**`bubble` is the canonical fork mode.** A fork is mid-conversation in the parent's terminal; permission prompts must appear where the user is, not in some headless worker process the user doesn't see.

**`plan` is the canonical research-worker mode.** A research worker that needs to grep, read, and explore but must not mutate state is a perfect plan-mode candidate. The framework structurally prevents writes; the worker cannot escape even if confused.

**Sub-agents inherit the parent's working-directory scope unless explicitly extended.** A fork running in `/repo/src` cannot wander into `/repo/secrets` even if the parent could, because the fork inherits the parent's `additionalWorkingDirectories` set.

---

## Pattern 6: Worktree Isolation for Speculative Writes

A worker that will modify the filesystem benefits from running in a **git worktree** — a separate checkout of the same repository, on a new branch, sharing the same `.git` directory but with isolated working files.

**Why worktrees beat subdirectories:**

- The worker can run `git status`, `git diff`, `git commit` against its branch without affecting the parent
- The parent keeps working in the original tree
- The worker's changes are auditable (diff the branch) and reversible (delete the branch)
- Multiple workers can have multiple worktrees on different branches, no contention

**The worker must be told it's in a worktree.** A worker that doesn't realize it's in a worktree will be confused by paths from the parent's context that don't exist in its own working tree. The framework injects a notice:

> "You've inherited the conversation context above from a parent agent working in `/parent/cwd`. You are operating in an isolated git worktree at `/worktree/cwd` — same repository, same relative file structure, separate working copy. Paths in the inherited context refer to the parent's working directory; translate them to your worktree root. Re-read files before editing if the parent may have modified them since they appear in the context. Your changes stay in this worktree and will not affect the parent's files."

This notice prevents two common failure modes: editing the wrong path, and reading stale file content from the parent's context instead of the current state in the worktree.

**Worktree cleanup is bounded by the work, not by the agent.** Don't clean up the worktree when the agent exits — clean up when the parent has consumed (merged or discarded) the work. Otherwise a successful worker's output gets wiped before the user sees it.

---

## Pattern 7: Inter-Agent Communication

Two communication primitives cover most needs:

**`SendMessage(to, message)`** — Direct message to one running agent. The recipient receives the message as a user-role input on its next turn. Use cases:

- Coordinator continues a worker after synthesis: "Now implement this spec: [...]"
- Coordinator stops a divergent worker: "Cancel that, the requirement changed"
- Worker pushes a status update back to the parent

The sender does not block on a reply. The recipient processes the message asynchronously.

**Team-shared message bus.** When more than two agents need to coordinate, a dedicated team has a named message channel. Any agent in the team can post to the channel; all agents see the messages. The bus persists across agent restarts.

Use the bus for:

- Cross-worker knowledge ("I found that the bug is in module X")
- Stop-the-world signals ("stop, the user changed their mind")
- Progress aggregation (workers post percent-complete; an aggregator updates the UI)

Avoid the bus for:

- Point-to-point work (just use `SendMessage`)
- High-frequency telemetry (the bus is for messages, not metrics)

---

## Pattern 8: Scratchpad — Durable Cross-Worker Knowledge

Messages are ephemeral. When workers need to share findings durably — across worker lifetimes, across restarts, beyond what fits in a message — the right primitive is a **shared scratchpad directory**.

The scratchpad is a per-session directory where any agent in the session can read and write without permission prompts. Conventions:

- One subdirectory per phase or per worker
- Markdown files for human-readable findings (the coordinator reads these)
- JSON files for structured data the next worker consumes
- A `_index.md` at the top describing the structure

**Scratchpad files are not the artifact.** They are notes. The final artifact (the PR, the report, the code) lives in the repo. The scratchpad is the working memory of the multi-agent system — when the work ships, the scratchpad can be archived or wiped.

**The scratchpad is bypass-immune.** Even in `bypassPermissions`, the framework does not auto-approve writes outside the scratchpad and the agreed working directories. A worker that tries to write to `/etc` from a scratchpad context still gets blocked.

---

## Pattern 9: Background Execution and Async Lifecycle

A worker's execution shape is one of:

- **Inline** — the parent waits in its current turn for the worker's reply. Simple, but the parent's turn is blocked.
- **Background** — the parent's call returns immediately with a task ID; the worker runs to completion in its own lifetime; the result arrives later as a notification.

**The decision rule.** Inline is fine for fast workers (< 30 s) and for workers whose result is needed before the parent can do anything else. Background is mandatory for workers whose runtime exceeds the parent's interactive patience, and is strongly preferred for any worker the user might want to cancel.

**Background execution decouples the request-response model.** The parent does not wait. It continues conversing with the user and gets notified when the worker finishes. The user can cancel the worker mid-flight without canceling the conversation.

The detailed lifecycle of background workers — task IDs, status polling, output streaming, persistence — is the subject of `10-task-lifecycle.md`.

**MaxTurns is the runaway floor.** Every agent definition declares a `maxTurns` — typically 50–200. When the worker hits this without completing, the framework terminates it. Without a limit, a confused worker can burn through dollars before anyone notices.

---

## Pattern 10: Remote Agents and the SDK Bridge

Some workers cannot run on the user's machine: they need credentials the user doesn't have, they take longer than the laptop will stay awake, they need to be reachable when the user is offline.

The pattern: **the agent runs in a remote service; the local client integrates with it via an RPC boundary.**

Three integration shapes are common:

- **Slack bridge.** The remote agent posts updates to Slack; the user replies in Slack; messages flow over a daemon to the agent.
- **PR bot.** The remote agent opens PRs; the user reviews them; comments become messages.
- **Webhook trigger.** External events (CI build complete, deploy finished) wake the agent up.

The remote agent is, structurally, just another sub-agent — but the spawning is `RemoteTrigger({ deployment, payload })` instead of `Agent({ subagent_type })`, and the result arrives via webhook instead of inline.

**Key constraint:** the remote agent has no shared context with the local client. Everything the remote needs must be in the trigger payload or in a shared store the remote can access. The remote cannot "see" the local user's filesystem.

---

## Pattern 11: Scheduling and Autonomous Loops

A scheduled agent is one that runs without a user prompting it. Triggers can be:

- **Cron-style** — "every Monday at 9 AM, run X"
- **Event-driven** — "when a PR is opened, run X"
- **Loop-based** — "every 15 minutes, check Y; stop when Z"

For each trigger, the framework needs to:

- Pick a fresh execution context (no stale state from a previous run)
- Establish permissions appropriate for an unattended run (`shouldAvoidPermissionPrompts = true`)
- Capture the result somewhere the user can review (file, ticket, message)
- Avoid loops — a scheduled agent that triggers itself recursively will be expensive

**The result must always be reviewable post-hoc.** An autonomous agent that does work and emits no record of doing it is auditing-hostile. Always write something — a log, a ticket, a Slack post, a file — so the user can verify what the agent did.

The detailed lifecycle of scheduled agents is in `10-task-lifecycle.md`; observability and cost controls for autonomous runs are in `11-observability.md`.

---

## Pattern 12: Concurrency Discipline

Multi-agent systems have new concurrency hazards. Three patterns help:

**Reads in parallel, writes in series, per scope.** Multiple workers can read the same files concurrently. Multiple workers writing the *same files* race; serialize them. Multiple workers writing *different files* are fine in parallel. The coordinator decides which workers can fan out and which must serialize.

**No worker checks on another worker.** When the coordinator wants to know whether worker A is done, it does NOT spawn worker B to ask worker A. It waits for the `task-notification` to arrive. Workers communicating with workers is a multiplier on overhead and a source of deadlocks.

**Stop signals propagate through the message bus, not by polling.** When the user cancels, the coordinator sends a stop message to every running worker. Workers do not poll a shared "should I stop?" flag — the cost of polling exceeds the cost of just receiving the message.

---

## Multi-Agent Design Checklist

**Topology selection**
- [ ] Do you have a per-task decision between fork, fresh, worker, team, remote, scheduled?
- [ ] Is the choice documented in the agent definition's `whenToUse` text?
- [ ] Are the right verification gates in place between phases (independent verifier agent)?

**Fork-specific**
- [ ] Does the fork thread the parent's rendered system prompt bytes, not call `getSystemPrompt()` again?
- [ ] Does the fork use `tools: ['*']` with `useExactTools: true`?
- [ ] Are placeholder tool_results byte-identical across all fork children?
- [ ] Is recursive fork prevented (boilerplate-tag detection in history)?
- [ ] Does the fork prompt forbid sub-agent spawning, conversation, and free-text preamble?
- [ ] Is the report format structurally parseable (labelled flat fields, not prose)?

**Coordinator-specific**
- [ ] Does the coordinator's prompt explicitly distinguish `<task-notification>` from user messages?
- [ ] Is `SendMessage` available for continuing existing workers?
- [ ] Are workers spawned in parallel where the work allows?
- [ ] Does the workflow have a Synthesis phase the coordinator owns?
- [ ] Does the coordinator avoid using one worker to check on another?

**Permission and isolation**
- [ ] Do forks use `bubble` mode for permission prompts?
- [ ] Do research-only workers use `plan` mode?
- [ ] Do speculative-write workers use a git worktree?
- [ ] Do workers in worktrees receive the worktree notice?

**Communication**
- [ ] Is there a `SendMessage` primitive for direct agent-to-agent communication?
- [ ] Is there a shared message bus for team-scope communication?
- [ ] Is there a scratchpad for durable cross-worker knowledge?

**Lifecycle**
- [ ] Does every agent definition declare `maxTurns`?
- [ ] Are background workers the default for long-running tasks?
- [ ] Are scheduled/autonomous agents required to leave a post-hoc auditable trail?

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Monolithic single agent for everything | Single point of failure, context bloat, no parallelism | Decompose into specialized agents |
| Fork that re-renders its system prompt | Prefix cache miss on every spawn | Thread `renderedSystemPrompt` from parent |
| Fork that allows recursive forking | Exponential blow-up; the user's first ask spawns 50 children | Boilerplate-tag check at spawn time |
| Fork prompt that doesn't forbid sub-agent spawning | Forks fork forks; runaway cost | Explicit "you ARE the fork, do not spawn" in the boilerplate |
| Fresh subagent for parallel exploration | Cache miss on the prefix that was free with forks | Use fork for parallel; fresh for distinct work |
| Coordinator that calls Bash directly | Defeats the topology; coordinator's context now grows with execution noise | Coordinator only orchestrates; workers execute |
| Worker that reports in free prose | Coordinator can't parse, has to read the whole thing | Structured labelled-field report |
| One worker checking on another | Extra round trips, deadlock risk | Wait for task-notification |
| Workers without `maxTurns` | Runaway loops, dollar burns | Always set `maxTurns`; default to a low number |
| Speculative writes in the parent's worktree | Parent and worker race | Spawn worker in a new worktree |
| Scratchpad as the final artifact | Work disappears when session ends | Scratchpad is working memory; artifact lives in repo |
| Remote agent that needs local-only context | RPC fails to deliver what the remote needs | Push all required context into the trigger payload |
| Scheduled agent with no audit trail | User cannot tell what the agent did or whether it was correct | Always write a reviewable record |
| Workers communicating via shared mutable state | Race conditions, lost updates | Messages or scratchpad, never shared in-memory state |
| Verifier inherits parent's reasoning | Verification is biased; "I already think this is right" | Verifier is a fresh subagent with no parent history |
