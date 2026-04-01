---
name: claude-code-methodology
description: Production-grade Agent development methodology extracted from Claude Code. 7-dimension framework covering tool design, system prompts, permission & safety, multi-agent orchestration, token economy, memory/state, and extensibility. Supports architecture design, implementation guidance, and agent review. Trigger on "Agent design", "build an agent", "AI agent", "tool design", "system prompt architecture", "agent review", "multi-agent", or any agent development concern.
---

# Agent Development Methodology

## Extracted from Claude Code

All patterns in this skill are derived from Claude Code. This is not theoretical — every pattern shown has been validated at scale in one of the most-used AI coding tools.

---

## Detect Mode and Jump In

Identify which mode the user needs, then work through it directly:

| Mode               | User Context                                                                | Action                                                                               |
| ------------------ | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| **Architecture**   | "I want to build an agent that..." / designing from scratch                 | Gather requirements → work through relevant dimensions → produce architecture design |
| **Implementation** | Specific question: "How do I design X?" / "What's the best approach for Y?" | Identify dimension → load reference file → apply pattern to their context            |
| **Review**         | Sharing existing code or design for critique                                | Read their work → evaluate each relevant dimension → structured review output        |

---

## The 7 Dimensions

Every production agent design should consider these dimensions. Not all apply equally — use judgment about which are critical for the user's use case.

| #   | Dimension                      | Core Question                                                 | Reference File                       |
| --- | ------------------------------ | ------------------------------------------------------------- | ------------------------------------ |
| 1   | **Tool Design**                | Are tools safe, well-described, and composable?               | `references/01-tool-design.md`       |
| 2   | **System Prompt Architecture** | Is the prompt maintainable, efficient, and layered?           | `references/02-system-prompt.md`     |
| 3   | **Permission & Safety**        | Is the agent fail-closed? Can it be trusted?                  | `references/03-permission-safety.md` |
| 4   | **Multi-Agent Orchestration**  | Is delegation, isolation, and parallelism well-designed?      | `references/04-multi-agent.md`       |
| 5   | **Context & Token Economy**    | Is context used efficiently throughout the agent's lifecycle? | `references/05-token-economy.md`     |
| 6   | **Memory & State**             | How is knowledge persisted across turns and sessions?         | `references/06-memory-state.md`      |
| 7   | **Extensibility**              | Can the agent be extended without modifying core logic?       | `references/07-extensibility.md`     |

---

## Mode 1: Architecture Design

When designing a new agent, work through this sequence:

### Step 1 — Understand the agent's purpose and boundaries

Ask or infer:

- What actions can the agent take? (→ determines tool inventory)
- What can go wrong? What's the blast radius of a mistake? (→ determines permission model)
- Who are the users and how much should the agent trust them? (→ determines default permission mode)
- Does it need memory across sessions? (→ determines state management)
- Will it ever delegate work to sub-agents? (→ determines orchestration needs)
- What's the expected conversation length / context size? (→ determines token management strategy)

### Step 2 — Identify which dimensions are critical

| Agent Type                | Usually Critical | Often Relevant | Rarely Needed |
| ------------------------- | ---------------- | -------------- | ------------- |
| Simple task agent         | 1, 2, 3          | 5, 6           | 4, 7          |
| Long-running coding agent | 1, 2, 3, 5, 6    | 4, 7           | —             |
| Multi-agent system        | 1, 2, 3, 4       | 5, 6, 7        | —             |
| Production API agent      | All 7            | —              | —             |

### Step 3 — Load relevant reference files and apply patterns

Read the relevant `references/` files for the critical dimensions and translate the patterns to the user's tech stack and constraints.

### Step 4 — Produce an architecture design document

Structure the output as:

```
## Agent Architecture: [name]

### Tool Inventory
[list tools with safety classification: read-only / write / destructive]

### System Prompt Structure
[static sections vs dynamic sections, what goes where]

### Permission Model
[default mode, escalation path, deny rules]

### State Management
[what's in-session vs persistent, memory layers]

### Multi-Agent Topology (if applicable)
[orchestrator/worker pattern, fork vs fresh, isolation]

### Token Budget Estimate
[context window usage, caching strategy]

### Extensibility Points
[how can this agent be extended or integrated]
```

---

## Mode 2: Implementation Guidance

When user has a specific implementation question:

1. **Map the question to a dimension** — e.g., "how do I truncate tool results?" → Dimension 5 (Token Economy)
2. **Read the relevant reference file** for that dimension
3. **Apply the pattern to their context** — adapt the Claude Code approach to their tech stack (Python, LangChain, CrewAI, custom, etc.) and their specific constraints
4. **Explain the WHY** — don't just say what to do, explain why Claude Code does it this way and what problem it solves

---

## Mode 3: Agent Review

When user shares existing agent code or design for review:

1. **Read their code/design** — understand what's there before evaluating
2. **Evaluate relevant dimensions** — not all 7 need to be in every review; focus on what matters for their agent type
3. **Prioritize findings**:
   - 🔴 **Critical**: Safety/correctness issues (e.g., no permission system, destructive tools with no guardrails)
   - 🟡 **High**: Architectural problems that will hurt later (e.g., monolithic system prompt, no context management)
   - 🟢 **Low**: Optimization opportunities (e.g., prompt caching not used)

4. **Output structured review**:

```
## Agent Review

### 🔴 Critical Issues
- [issues that could cause harm, data loss, incorrect behavior, or security problems]

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

## Core Principles (from Claude Code's Design Philosophy)

These are the highest-level philosophies distilled from the source. Apply these as a sanity-check on any agent design:

1. **Fail-closed over fail-open** — When permission is ambiguous, deny by default. Safety must be explicitly granted, not assumed.

2. **Token economics drive architecture** — Every structural decision (tool loading, prompt sections, result storage) should consider token cost. Bad token economics don't matter at 100 users; they matter a lot at 100K.

3. **Static content and dynamic content must be separated** — Stable instructions belong in cacheable sections. Turn-specific data (env info, memory, user config) belongs in volatile sections. Mixing them wastes cache.

4. **Tools define capability boundaries** — The tool set IS the agent's capability model. What isn't a tool can't be done. This makes the agent's abilities explicit and auditable.

5. **Extensibility through protocols, not inheritance** — Claude Code uses MCP (protocol), Hooks (events), and Skills (user-defined workflows) — never subclasses its way to extensibility. Prefer standard interfaces.

6. **Recursive delegation is a design primitive** — Agents can spawn agents. If you design without this, you'll hit ceilings. Design the tool interface as if sub-agents might call it too.

7. **The permission model is a trust spectrum** — `default → plan mode → bypass` is a dial, not a switch. Design the permission system as a spectrum from cautious to autonomous.
