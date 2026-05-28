---
name: claude-code-methodology
description: Production-grade Agent development methodology derived from Claude Code. 11-dimension framework covering tool design, system prompts, permissions, multi-agent orchestration, token economy, memory, extensibility, conversation loop, workflow modes, task lifecycle, and observability. Supports architecture design, implementation guidance, and agent review. Trigger on "Agent design", "build an agent", "AI agent", "tool design", "system prompt architecture", "agent review", "multi-agent", "compaction", "agent workflow modes", "background tasks", "agent observability", or any agent development concern.
---

# Agent Development Methodology

## Production-grade patterns from a real coding agent

All patterns in this skill are derived from a production coding agent at scale. They are not theoretical — every pattern shown is validated by use against real engineering problems by real engineers. The patterns are framework-agnostic; the dimensions, the trade-offs, and the discipline transfer to any technology stack.

---

## Detect Mode and Jump In

Identify which mode the user needs, then work through it directly:

| Mode | User context | Action |
|---|---|---|
| **Architecture** | "I want to build an agent that..." / designing from scratch | Gather requirements → work through relevant dimensions → produce architecture design |
| **Implementation** | "How do I design X?" / "What's the best approach for Y?" | Identify dimension → load reference file → apply pattern to their context |
| **Review** | Sharing existing code or design for critique | Read their work → evaluate each relevant dimension → structured review output |

---

## The 11 Dimensions

A production agent should be evaluated along these dimensions. Not all apply equally for every project — but every production design touches at least seven, and ignoring any one of them creates a class of failures that the dimension would have prevented.

| # | Dimension | Core question | Reference |
|---|---|---|---|
| 1 | **Tool Design** | Are tools safe, well-described, composable, and self-declarative? | `references/01-tool-design.md` |
| 2 | **System Prompt Architecture** | Is the prompt modular, layered, and cache-disciplined? | `references/02-system-prompt.md` |
| 3 | **Permission and Safety** | Is the agent fail-closed, with structured decisions and an operator ceiling? | `references/03-permission-safety.md` |
| 4 | **Multi-Agent Orchestration** | Is the right topology used per task (fork/fresh/worker/team/remote/scheduled)? | `references/04-multi-agent.md` |
| 5 | **Context and Token Economy** | Are the four token pools managed with the right strategies (cache, defer, compact, evict)? | `references/05-token-economy.md` |
| 6 | **Memory and State** | Are the five state categories distinguished, and is persistent memory file-based and triggered? | `references/06-memory-state.md` |
| 7 | **Extensibility** | Can hooks, skills, MCP, and plugins extend the agent without modifying it? | `references/07-extensibility.md` |
| 8 | **Conversation Loop** | Is the loop streaming, interruptible, retry-aware, and invariant-preserving? | `references/08-conversation-loop.md` |
| 9 | **Workflow Modes** | Are there structural mode primitives (plan, worktree) for phase-level constraints? | `references/09-workflow-modes.md` |
| 10 | **Task Lifecycle** | Are background tasks first-class objects with status, output, cancellation, resume? | `references/10-task-lifecycle.md` |
| 11 | **Observability and Cost** | Is every interesting moment structured, attributed, and queryable? | `references/11-observability.md` |

---

## Mode 1: Architecture Design

When designing a new agent, work through this sequence:

### Step 1 — Understand the agent's purpose and boundaries

Gather (ask or infer):

- What actions can the agent take? (→ determines tool inventory and safety profiles)
- What can go wrong? What's the blast radius of a mistake? (→ determines permission model and modes)
- Who are the users and how much should the agent trust them? (→ determines default permission mode, audience variants)
- Does it need memory across sessions? (→ determines memory architecture)
- Will it delegate work to sub-agents? (→ determines multi-agent topology)
- What's the expected conversation length? (→ determines compaction strategy)
- Will it run autonomously or only interactively? (→ determines observability rigor, budget enforcement)
- Will it need to be extensible by third parties? (→ determines hook/skill/plugin design)

### Step 2 — Identify which dimensions are critical

| Agent type | Always critical | Often critical | Sometimes critical |
|---|---|---|---|
| **Simple task agent** | 1, 2, 3 | 5 | 6, 8 |
| **Long-running coding agent** | 1, 2, 3, 5, 6, 8, 9 | 4, 10, 11 | 7 |
| **Multi-agent / coordinator** | 1, 2, 3, 4, 8, 10 | 5, 6, 9, 11 | 7 |
| **Autonomous / scheduled** | 1, 2, 3, 10, 11 | 4, 5, 6, 8 | 7, 9 |
| **Plugin-extensible production** | All 11 | — | — |

### Step 3 — Load relevant reference files and apply patterns

Read the relevant `references/` files for the critical dimensions and translate the patterns to the user's stack and constraints. Each reference has a Design Checklist at the end — use it to verify your design.

### Step 4 — Produce an architecture design document

Structure the output as:

```
## Agent Architecture: [name]

### Tool Inventory
[list tools with safety profiles: read-only / write / destructive; per-input safety functions]

### System Prompt Structure
[static sections inventory; dynamic sections inventory; boundary marker; audience variants]

### Permission Model
[default mode; mode ladder; rule sources; bypass-immune ceiling; classifier policy]

### Memory and State
[session state shape; persistent memory directory layout; the four memory types in use]

### Multi-Agent Topology (if applicable)
[fork/fresh/worker/team/remote choices per task; worktree usage; coordinator workflow]

### Token Economy
[effective window calc; compaction strategy stack; cache strategy; tool result budget]

### Conversation Loop
[streaming approach; cancellation; retry policy; compact_boundary handling]

### Workflow Modes
[plan mode and exit deliverable; worktree mode if applicable; other modes if invented]

### Task Lifecycle
[background task types in use; concurrency limits; resume policy]

### Extensibility
[hook events surfaced; skill loading; MCP server inventory; plugin packaging if applicable]

### Observability
[per-turn metrics; cost attribution dimensions; budget enforcement; session summary]
```

---

## Mode 2: Implementation Guidance

When the user has a specific implementation question:

1. **Map to a dimension.** "How do I truncate tool results?" → Dimension 5. "How does fork share a cache?" → Dimension 4. "How do I structure my permission decisions?" → Dimension 3.

2. **Read the relevant reference file** for the dimension.

3. **Apply the pattern to their context.** Adapt the canonical pattern to their stack (Python/LangChain/CrewAI/custom) and constraints. Show them what the analogue looks like in their world.

4. **Explain WHY**, not just what. Each pattern has a problem it solves — explain the problem and how the pattern addresses it. Patterns without their reasoning don't transfer; patterns with their reasoning do.

---

## Mode 3: Agent Review

When the user shares existing agent code or a design for review:

1. **Read their code/design.** Understand what's there before evaluating.

2. **Evaluate relevant dimensions.** Not every review needs all 11; focus on what matters for their agent type and stated concerns.

3. **Prioritize findings:**
   - 🔴 **Critical** — safety/correctness issues that could cause real harm, data loss, security problems, or runaway cost
   - 🟡 **High** — architectural issues that will compound over time (monolithic prompt, no context management, no observability)
   - 🟢 **Low** — optimization opportunities (cache not used, defer not used, cost attribution missing)

4. **Output structured review:**

```
## Agent Review

### 🔴 Critical Issues
- [issues that could cause harm, data loss, security problems, or runaway cost]

### 🟡 Architectural Concerns
- [issues with structure that will compound over time]

### 🟢 Optimization Opportunities
- [efficiency, cost, latency improvements]

### ✅ What's Well-Designed
- [acknowledge genuinely good patterns — be specific]

### Recommended Changes (Priority Order)
1. [most impactful change]
2. ...
```

---

## Core Principles (from the design philosophy)

These are the highest-level invariants extracted across all 11 dimensions. Apply them as sanity checks on any agent design.

1. **Fail-closed over fail-open.** When permission is ambiguous, deny by default. When the classifier errors, deny. When the input is malformed, ask. Safety must be explicitly granted, not assumed.

2. **Token economics drive architecture.** Every structural decision — section boundaries, tool loading, result storage, fork strategy — should consider token cost. Bad economics matter at 100K users.

3. **Static and dynamic must be separated.** Stable instructions go in cacheable sections; volatile data goes in dynamic sections. Mixing them wastes cache and degrades silently.

4. **Tools define capability boundaries.** The tool set IS the agent's capability model. What isn't a tool can't be done. This makes abilities explicit and auditable.

5. **Extensibility through protocols and events, not inheritance.** Hooks (shell commands), Skills (markdown), MCP (RPC), Plugins (bundles) — never subclass.

6. **Recursive delegation is a design primitive.** Agents spawn agents; design every tool interface as if a sub-agent might also call it.

7. **The permission model is a spectrum, not a switch.** Default → plan → acceptEdits → bypassPermissions → auto is a dial users adjust based on trust and task; the ceiling is operator-controlled.

8. **Modes are structural constraints with deliverables.** Plan mode and worktree mode are first-class state transitions the model enters via tools, with concrete completion artifacts.

9. **Background work is a first-class primitive.** Tasks are persistent state, not transient processes. They survive restarts, deliver results asynchronously, and have observable lifecycles.

10. **Every interesting moment is structured.** Permission decisions, cost transitions, hook executions, compactions — all carry structured reasons. Logs are queryable; analytics are answerable.

11. **The agent should know its own state.** Cost trajectory, budget headroom, token pressure, recent failures — surface this in the model's context so the model can make cost-aware and pressure-aware choices.

---

## Worked Example: A "PR Review Agent" Across All 11 Dimensions

A reviewer agent that takes a PR URL and produces a structured review with priorities, security checks, and verification suggestions. We walk through how each dimension shapes the design.

**1. Tool Design.** The reviewer needs `gh_pr_view` (for PR metadata), `file_read` (for diffed files), `grep` (for finding callers), `web_search` (for security CVEs), and `submit_review` (the final action). Each tool declares per-input safety: `gh_pr_view` is read-only regardless of input; `submit_review` is destructive (publishes). The submit tool's prompt explicitly says WHEN NOT TO USE — "do not submit until you have a complete review with priorities and at least one verification recommendation per change category."

**2. System Prompt.** The static section defines the reviewer's persona ("you find bugs that would ship to production"), the review structure (intro / per-file / synthesis), and the boundary between observed and inferred. The dynamic section carries the PR's metadata (URL, author, base branch, current date) and the user's review depth preference (cursory / standard / paranoid).

**3. Permission and Safety.** The reviewer runs in **plan mode** by default — it reads but cannot mutate the repo. The `submit_review` tool is the only write and requires user confirmation regardless of mode. The classifier policy is "the reviewer should never auto-approve writes; it should always ask."

**4. Multi-Agent.** For large PRs (>20 files), the main reviewer spawns **fork sub-agents** to review file groups in parallel — they share the parent's prompt cache (cheap), produce structured per-file reports, and the main reviewer synthesizes. For security review specifically, it spawns a **fresh subagent** with a hardened security prompt (not biased by the main reviewer's "looks fine" conclusions).

**5. Token Economy.** The PR diff itself can be large; results are subject to per-tool truncation. The conversation-level result budget evicts oldest file contents after synthesis is complete — the reviewer only needs the recent files for follow-up Q&A. Forks share prompt cache; the cache is the difference between affordable and not at scale.

**6. Memory and State.** The reviewer extracts feedback memories: "this team prefers integration tests over unit tests for service layer", "team blocks PRs without RFC for any DB schema change". On the next review, these are auto-loaded; new reviews get more accurate over time.

**7. Extensibility.** A `PreToolUse` hook on `submit_review` can run the team's CI lint as a final check; a custom skill `/security-deep` can switch the reviewer into paranoid mode; an MCP server can add a `query_jira` tool for linking issues.

**8. Conversation Loop.** The loop must handle the user interrupting mid-review ("actually focus on auth") — file reads are `cancel`-on-interrupt, the synthesis tool is `block`-on-interrupt (so it finishes the file it's processing). Streaming events show the reviewer's progress through files.

**9. Workflow Modes.** Plan mode is the default. A separate **deep-investigation** mode (custom) might be added: when entered, the reviewer disables network tools and is allowed only the local code+tests, to focus the investigation.

**10. Task Lifecycle.** Sub-agent forks for parallel file review are **background tasks**: the main reviewer spawns them, continues with synthesis prep, and consumes their task-notifications as they complete. If the user cancels, all forks are stopped and their partial reports preserved.

**11. Observability.** Per-review cost is tracked; the reviewer surfaces "this review used $0.42, 3 sub-agents" in the final report. A budget cap (`maxBudgetUsd: 2.00`) prevents one weird PR from consuming a day's budget. The diagnostic event stream records every file read, every classifier decision, every hook execution — letting the team analyze "why was the reviewer slow on PR #1234?"

The 11 dimensions are not separable in practice — they reinforce each other. A reviewer that gets just one dimension right (say, beautiful tool design) but ignores the others ends up unusable. A reviewer that addresses all 11 ships and improves over time.
