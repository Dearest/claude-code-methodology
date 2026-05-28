# Task Lifecycle Patterns

---

## The Core Insight

A **task** is a unit of work the agent can launch, observe, continue, and stop independent of the conversation's foreground turn. Tasks decouple "what the user is asking right now" from "what the agent is doing in parallel." Without them, every long-running action blocks the conversation; with them, the agent can launch ten things and stay responsive.

A production agent does not have one kind of task — it has several, each with different runtime characteristics, lifecycle hooks, and integration points:

| Task type | Runtime | Permission context | Use case |
|---|---|---|---|
| **LocalShell** | Spawned subprocess | Inherits agent's shell permissions | Long-running build, test, dev server |
| **LocalAgent** | Sub-agent on same machine | Per-agent definition | Worker in coordinator topology, background subagent |
| **RemoteAgent** | Sub-agent on remote service | Per-deployment policy | Long-running work that survives local machine shutdown |
| **InProcessTeammate** | Sub-agent sharing the parent's session | Shares parent | Multi-agent collaboration in one session |
| **LocalWorkflow** | Multi-step workflow | Composes other tasks | Pipelines: "research → plan → implement → verify" |
| **MonitorMcp** | Long-running MCP server probe | Per-server config | Health checks, recurring queries |
| **Dream** | Autonomous loop with self-pacing | Designated dream policy | Autonomous background exploration |

The patterns below treat the task as a first-class primitive with its own lifecycle, not as a special case of agent invocation.

---

## Pattern 1: Task as Structured State

A task is not "a process I started" or "a subagent I spawned" — it is a persistent state object the framework manages:

```typescript
type TaskState = {
  id: string                    // stable identifier
  type: TaskType                // discriminator: 'shell' | 'agent' | 'remote' | ...
  status: 'pending'             // queued, not started
        | 'running'             // active
        | 'completed'           // finished successfully
        | 'failed'              // finished with error
        | 'killed'              // stopped by user or system
  isBackgrounded: boolean       // foreground (inline) vs background (async)

  // Provenance
  createdAt: timestamp
  parentTaskId?: string         // for nested tasks
  spawnedBy: 'user' | 'agent' | 'hook' | 'schedule'

  // Identification for UI
  label: string                 // human-readable name
  pillLabel: string             // very short UI badge

  // Execution
  input: TaskInput              // initial parameters, varies by type
  output?: TaskOutput           // populated on success
  error?: TaskError             // populated on failure

  // Lifecycle
  startedAt?: timestamp
  finishedAt?: timestamp

  // Streaming
  outputPath?: string           // where streaming output is persisted
  lastProgressAt?: timestamp
}
```

**Why a structured state object?**

- The UI can render any task's status, regardless of type
- The user can list, filter, query, and reference tasks by ID
- A crashed process can be re-attached by reading its state from disk
- Telemetry aggregates over tasks consistently

---

## Pattern 2: Background vs Foreground Distinction

The same task type can run in foreground or background mode:

- **Foreground** — the spawning model call blocks on the task's completion; the result is returned synchronously to the model
- **Background** — the spawning model call returns immediately with a task ID; the task runs to completion in its own lifetime; the result arrives later as a notification

The `isBackgrounded` field on the task state distinguishes them. Tasks start foreground by default; they migrate to background when:

- A configurable timeout fires (typically 120 seconds)
- The user explicitly backgrounds the task
- The spawn was async-only by definition (e.g., remote tasks)

**The migration matters for UX.** A foreground task that took 30s is fine — the user waits and gets the result. A foreground task that takes 5 minutes is broken — the user is staring at a spinner unable to do anything else. Auto-migration after a threshold turns the second case into a usable background workflow.

**The background tasks indicator.** The UI surfaces all running background tasks in a persistent indicator (badge with count, list on click). This gives the user constant awareness of "what is the agent doing right now?" without intrusive notifications.

---

## Pattern 3: Spawn → Stream → Complete Lifecycle

Every task follows the same lifecycle:

```
spawn  →  pending  →  running  →  (completed | failed | killed)
                          ↓
                       stream output
                          ↓
                       update lastProgressAt
```

**Spawn** is the act of creating the task state. The framework allocates an ID, persists the initial state, returns the ID to the caller. The actual execution may or may not have started yet.

**Pending** means queued, not running. Tasks can be in pending state if:

- Concurrency limits cap how many tasks run simultaneously
- A dependency is not yet met
- A scheduling hook delays execution

**Running** means actively executing. The task may be streaming output. The framework records `lastProgressAt` on every progress event so the UI can show staleness ("no progress for 2 min").

**Completed / failed / killed** are the terminal states:

- `completed` — execution finished successfully; `output` is populated
- `failed` — execution finished with error; `error` is populated; output may be partial
- `killed` — execution stopped by user or system; partial output preserved

Once terminal, a task's state is immutable. Querying its result returns the cached state.

---

## Pattern 4: Output Streaming and Persistence

For background tasks, the model's turn returns long before the task completes. Output must be:

- Streamed live (so the UI can show progress)
- Persisted to disk (so the agent can read it later without bloating context)
- Bounded (so a runaway task doesn't fill the disk)

The pattern: **per-task output file with size cap.**

```python
class TaskOutputManager:
    def __init__(self, task_id):
        self.path = task_output_path(task_id)
        self.size = 0
        self.cap = TASK_OUTPUT_MAX_BYTES  # e.g., 50 MB

    def append(self, chunk):
        if self.size + len(chunk) > self.cap:
            self._truncate_to_keep_recent()
        self._write_chunk(chunk)
        self.size += len(chunk)
        self._update_last_progress_at()
```

**Truncation strategy.** When a task's output exceeds the cap, keep the most recent N MB and drop the oldest. The model is more often interested in "what is it doing now?" than "what did it do an hour ago." For high-value tasks where the early output matters (test runs), the strategy can be middle-truncate (keep start and end, elide middle).

**Reading task output from the model.** A `TaskOutput` tool lets the model read the current state of a task's output:

```
TaskOutput({ taskId: "abc123", offset: 0, limit: 200 })
```

This returns the file content (paged), not the full output. The model can navigate through long outputs without loading the whole thing into context.

---

## Pattern 5: Task Notification Envelope

When a background task completes (or fails), the framework needs to deliver the result back to the parent agent. The mechanism: a **task-notification** synthetic user message in the parent's next turn.

```xml
<task-notification>
  <task-id>abc123</task-id>
  <status>completed|failed|killed</status>
  <summary>Investigate auth bug</summary>
  <result>Found null pointer in src/auth/validate.ts:42...</result>
  <usage>
    <total_tokens>15234</total_tokens>
    <tool_uses>23</tool_uses>
    <duration_ms>45230</duration_ms>
  </usage>
</task-notification>
```

The parent's system prompt teaches it to recognize this envelope as a task result, not a real user message. The parent reads the `<result>`, acts on it, and continues.

**The envelope is a user message, not an assistant message.** This is deliberate: it lets the parent respond to the task result with its own next assistant turn. If results came as assistant messages, the parent would have to immediately turn around and produce another assistant message, doubling the turn count.

**Multiple tasks can complete in one turn.** If three tasks finish between the parent's turns, the parent's next user message contains three `<task-notification>` blocks, one per finished task.

---

## Pattern 6: Task Cancellation

The `TaskStop` tool lets the parent kill a task before it completes:

```
TaskStop({ taskId: "abc123", reason: "User changed their mind" })
```

The framework:

1. Sends an abort signal to the task's execution context (kill the subprocess, abort the sub-agent's loop, signal the remote service)
2. Updates the task state to `killed`
3. Preserves whatever output has been collected so far
4. Emits a `TaskCompleted` hook (with `status: killed`)
5. Delivers a notification to the parent on its next turn

**Cancellation is best-effort, not synchronous.** The parent's `TaskStop` call returns immediately. The actual termination may take a few seconds (waiting for the subprocess to clean up, the sub-agent to finish its current API call, the remote service to acknowledge). The parent sees the final `killed` notification when termination is confirmed.

**Killed task output is still valuable.** Don't discard a killed task's accumulated output. The user often kills a task to inspect what it found before deciding what to do next. The `TaskOutput` tool still returns the killed task's output.

---

## Pattern 7: Task Continuation via SendMessage

Tasks that wrap conversational sub-agents (LocalAgent, RemoteAgent, InProcessTeammate) can be **continued**, not just spawned. The `SendMessage` tool sends a new message into the sub-agent's conversation:

```
SendMessage({ to: "agent-a1b", message: "Now do part two: ..." })
```

The sub-agent receives the message as a new user input, processes it, and produces another assistant turn. The result arrives via another task-notification.

**Why continuation matters.** The sub-agent's loaded context (memory, file cache, knowledge of the codebase) is expensive to build. Throwing it away after one task and spawning a fresh agent for the next task wastes work. Continuation reuses the loaded context.

**The agent's identity persists.** The `task-id` field is also the agent's identity for `SendMessage`. The parent uses the same ID across many `SendMessage` calls; the agent retains its conversation history across the calls.

**Restarting a continued agent.** If a continued agent dies (machine restart, network failure), the framework can spin up a new instance and replay the message history. The agent ID is durable; the runtime backing it is replaceable.

---

## Pattern 8: Task Concurrency and Resource Limits

A naive system that launches every task immediately runs out of resources:

- Local processes consume RAM, CPU, file descriptors
- Local agents consume API budget per call
- Remote agents consume cloud quota
- Shell tasks compete for terminal stdin

The pattern: **explicit concurrency limits per task type.**

```
max_concurrent_local_shell:   8
max_concurrent_local_agent:   5
max_concurrent_remote_agent:  20
max_concurrent_in_process:    3
```

Tasks beyond the limit go into `pending` state until a running task finishes. The framework's scheduler picks pending tasks in FIFO order.

**The limits are operator-tunable.** A power user with a big machine can raise local limits; a hosted deployment can set tighter limits to control cost. The defaults should be conservative.

**Per-resource categories.** Limits are per task type, not per parent. Otherwise a single parent could starve siblings by launching 100 local-agent tasks. The agent-fleet-level limit ensures fair sharing.

---

## Pattern 9: Resume After Restart

When the agent process restarts (crash, machine reboot, intentional shutdown), tasks should not all just disappear. The persistence model:

- **Task state is in durable storage** (sqlite, files) updated on every state change
- **At startup, the framework enumerates pending and running tasks** from storage
- **For each, it decides:**
  - The runtime is recoverable (subprocess pid still alive, remote agent contactable) → attach
  - The runtime is gone but the work is idempotent → restart
  - The runtime is gone and restart is unsafe → mark `failed` with a "lost runtime" reason

**The recoverability decision is per task type.** A local shell with a known pid is recoverable if the pid is still owned by the same process. A remote agent is recoverable if the deployment endpoint says the agent is alive. A local agent in a separate process is recoverable if that process is alive.

**User-facing behavior on resume.** Show the user "5 tasks recovered, 2 lost" and let them decide whether to retry the lost ones. Silent loss erodes trust.

---

## Pattern 10: Scheduled and Recurring Tasks

A task can be:

- **Once** — runs at a specific time, then never again
- **Cron-style** — runs on a schedule (every Monday at 9 AM)
- **Loop** — runs repeatedly with self-pacing (re-fire every N seconds, possibly variable)
- **Event-driven** — runs on an external event (webhook, file change, MCP notification)

For each, the framework needs:

- A **trigger registry** that knows when to fire
- A **fresh execution context** per fire (no stale state from previous run)
- **Run identifiers** that distinguish fires of the same scheduled task
- A **result destination** other than the user's current conversation (file, ticket, message channel)

The scheduled task is **not in the user's conversation.** A user who didn't initiate the run shouldn't be surprised by it. Results land in a designated location the user can review on their own time.

**Loops have a special primitive: self-pacing.** A loop fires once, decides how long to sleep before the next fire, schedules itself, and exits. The framework wakes it up at the scheduled time. This lets the agent adaptively pace itself — sleep longer when nothing's changing, wake faster when there's activity.

---

## Pattern 11: Dream Tasks — Autonomous Idle Work

A dream task is a long-running autonomous loop that runs while the user is away. Examples:

- "Explore the codebase and write notes about architecture you find interesting"
- "Watch the build and report any new failures"
- "Cleanup loose ends from the last session"

Dream tasks share the constraint set of all autonomous runs:

- `shouldAvoidPermissionPrompts = true` (no human to ask)
- `maxBudgetUsd` set (no unbounded cost)
- Result lands in a structured location (file, ticket)
- Audit trail of every action

**Dream-specific behaviors:**

- The dream task expects to be interrupted at any moment (the user comes back) — its state must be checkpointable
- The dream task self-limits its scope (works on bounded sub-problems, not the whole project at once)
- The dream task produces value even if killed at 10% completion — partial work is useful

The dream pattern is the upper bound of agent autonomy. Used well, it makes the agent useful while the user is doing other things. Used badly, it burns through the budget on undirected exploration.

---

## Pattern 12: Task UI Surface

The user needs to see what's running, what just finished, what failed. A coherent surface:

- **Background tasks indicator** — persistent badge with running task count; click to see the list
- **Per-task status row** — label, status, runtime, last progress timestamp
- **Detailed task view** — full output (paged), parameters, parent, related tasks
- **Notification on terminal state** — desktop/OS notification when a task completes or fails (configurable)

**Notifications respect user attention.** A user actively typing should not be interrupted by a "task X completed" toast. Surface the result in the persistent indicator; show toast only on failures or on explicit "notify when done" tasks.

**Tasks have human-readable labels and short pill labels.** The full label ("Investigate authentication bug in login flow") shows in the detailed view; the pill ("Auth bug") shows in the compact indicator. The framework generates the pill from the label when not provided.

---

## Task Lifecycle Design Checklist

**State management**
- [ ] Is the task state a structured object, not a process handle?
- [ ] Is the state persisted on every transition?
- [ ] Are states drawn from a small enum (pending, running, completed, failed, killed)?
- [ ] Is the task type a discriminator that lets the UI render any task generically?

**Background vs foreground**
- [ ] Is there an `isBackgrounded` field distinguishing modes?
- [ ] Is auto-migration to background triggered by timeout?
- [ ] Is the background indicator surfaced persistently in the UI?

**Output management**
- [ ] Is task output streamed to a per-task file with a size cap?
- [ ] Is truncation strategy appropriate per task type (recent-keep, middle-elide)?
- [ ] Is there a `TaskOutput` tool that pages output without loading it whole?

**Notification flow**
- [ ] Are task completions delivered as task-notification synthetic user messages?
- [ ] Are multiple completions in one turn merged into one user message with multiple blocks?
- [ ] Does the parent's prompt teach it to recognize the envelope?

**Cancellation**
- [ ] Is there a `TaskStop` tool that signals abort?
- [ ] Does cancellation preserve accumulated output?
- [ ] Is termination best-effort with delivery of the final `killed` notification?

**Continuation**
- [ ] Are conversational sub-agent tasks continuable via `SendMessage`?
- [ ] Does the task ID also serve as the agent's identity for continuation?
- [ ] Can the framework restart a continued agent that died?

**Concurrency**
- [ ] Are there explicit per-task-type concurrency limits?
- [ ] Are limits operator-tunable?
- [ ] Is task scheduling FIFO with pending state for queued tasks?

**Resilience**
- [ ] At startup, does the framework enumerate and re-attach pending/running tasks?
- [ ] Is the recoverability decision documented per task type?
- [ ] Are lost-runtime tasks marked `failed` with a clear reason?

**Scheduling**
- [ ] Is there a trigger registry for cron/once/loop/event tasks?
- [ ] Are scheduled task results delivered to a destination other than the user's conversation?
- [ ] Are loops self-paced with explicit wake-up scheduling?

**Autonomous (dream) tasks**
- [ ] Are dream tasks bounded by `maxBudgetUsd` and `shouldAvoidPermissionPrompts`?
- [ ] Do they produce reviewable audit trails?
- [ ] Can they checkpoint state for interruption?

**UI**
- [ ] Is the background task indicator persistent?
- [ ] Are notifications on completion configurable?
- [ ] Do tasks have both detailed labels and short pill labels?

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Task = pid | No persistence, no resume, no rich state | Task = structured state object with status, output, error |
| All tasks foreground | 5-minute tasks block the conversation | Auto-migrate to background after timeout |
| Background task with no UI indicator | User loses track of what's running | Persistent background tasks badge |
| Unbounded output | Disk fills; long task crashes the agent | Per-task size cap with truncation |
| Output streamed only to log files | UI can't show progress; user feels nothing is happening | Stream to a file the framework reads + emits as progress events |
| Task result returned synchronously to the spawn call | Parent's turn blocks; loses parallelism | Spawn returns ID immediately; result arrives later as notification |
| Notifications as assistant messages | Parent forced into a self-reply chain | Notifications as synthetic user messages |
| `TaskStop` discards output | User loses the partial work they wanted to inspect | Killed tasks preserve output |
| New agent for every continuation | Throws away expensive loaded context | `SendMessage` to existing task ID |
| No concurrency limit | OOM, rate-limited, system thrashes | Per-task-type concurrency caps |
| No resume after restart | Pending tasks lost on crash | Persist state durably; enumerate on startup |
| Scheduled task results delivered to current conversation | User confused by unexpected messages | Dedicated destination (file, ticket, channel) |
| Dream task with no budget cap | Burns through dollars unattended | `maxBudgetUsd` required for autonomous tasks |
| Dream task with no audit trail | User can't tell what the agent did or whether it was right | Always write a reviewable record |
| Toast on every task completion | Notification spam | Configurable: noisy events for failures, quiet for completions |
| Same label everywhere (full sentence in pill, full sentence in detailed view) | Pill too long, detailed view too short | Separate pill label from full label |
