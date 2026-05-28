<div align="center">

# Claude Code Methodology

> _"Build your agents the way production agents are built."_

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-blueviolet)](https://claude.ai/code)
[![AgentSkills](https://img.shields.io/badge/AgentSkills-Standard-green)](https://agentskills.io)

<br>
Every tool call, every permission check, every multi-agent delegation, every compaction —<br>
is a design decision battle-tested at scale.<br>

**Distilled into 11 dimensions, ready to apply to any agent project.**

<br>

From tool design to token economy, from permission models to multi-agent orchestration,<br>
from conversation loop to task lifecycle, from workflow modes to observability —<br>
covering the full lifecycle of agent development

[Install](#install) · [Usage](#usage) · [11 Dimensions](#11-dimensions) · [When to Use](#when-to-use) · [**中文**](README.md)

</div>

---

## What is this

An **Agent development methodology** extracted from a production-scale agent, packaged as a Claude Code Skill.

Every pattern and anti-pattern is derived from a real production agent — not theory. Each one has been validated at scale, and the patterns are framework-agnostic: they transfer to Python, Go, Rust, LangChain, CrewAI, or any custom stack.

---

## Install

```bash
npx skills add Dearest/claude-code-methodology
```

Once installed, the skill automatically activates in any Claude Code conversation involving agent development.

---

## Usage

After installation, simply describe your agent development needs in Claude Code:

- **"Help me design an agent that auto-fixes bugs"** → Architecture Design mode
- **"Review the tool design of this agent"** → Agent Review mode
- **"My agent is consuming too many tokens"** → Token Economy guidance
- **"How do I delegate tasks in a multi-agent setup?"** → Orchestration guidance
- **"My context keeps overflowing, how should I do compaction?"** → Conversation Loop + Token Economy
- **"How should I manage background tasks?"** → Task Lifecycle
- **"I want to build an autonomous agent — how do I control cost?"** → Observability

### Three Modes

**1. Architecture Design** — Design a new agent from scratch. Walk through requirements, analyze by dimension, produce a full architecture document.

**2. Implementation Guidance** — For specific questions, map to the right dimension, explain a battle-tested approach with reasoning, adapted to your stack.

**3. Agent Review** — Structured review of existing agent code or design, evaluated by dimension, with prioritized findings and recommendations.

---

## 11 Dimensions

| #   | Dimension                              | Core Question |
| --- | -------------------------------------- | ------------- |
| 1   | **Tool Design**                        | Are tools safe, self-declarative, and composable? |
| 2   | **System Prompt Architecture**         | Is the prompt modular, layered, and cache-disciplined? |
| 3   | **Permission & Safety**                | Is the agent fail-closed, with structured decisions and an operator ceiling? |
| 4   | **Multi-Agent Orchestration**          | Is the right topology used per task (fork/fresh/worker/team/remote/scheduled)? |
| 5   | **Token Economy**                      | Are the four token pools managed with the right strategies (cache, defer, compact, evict)? |
| 6   | **Memory & State**                     | Are the five state categories distinguished, and is persistent memory file-based and triggered? |
| 7   | **Extensibility**                      | Can hooks, skills, MCP, and plugins extend the agent without modifying it? |
| 8   | **Conversation Loop**                  | Is the loop streaming, interruptible, retry-aware, and invariant-preserving? |
| 9   | **Workflow Modes**                     | Are there structural mode primitives (plan, worktree) for phase-level constraints? |
| 10  | **Task Lifecycle**                     | Are background tasks first-class objects with status, output, cancellation, resume? |
| 11  | **Observability & Cost**               | Is every interesting moment structured, attributed, and queryable? |

Each dimension has its own reference document with patterns, anti-patterns, design checklists, and adaptation examples.

---

## Core Principles

These are the highest-level invariants extracted across all 11 dimensions:

1. **Fail-closed over fail-open** — Deny by default when permission is ambiguous; deny when classifier errors; ask when input is malformed
2. **Token economy drives architecture** — Every structural decision should account for token cost
3. **Separate static and dynamic content** — Stable instructions in cacheable sections; volatile data in dynamic sections
4. **Tools define capability boundaries** — The toolset IS the agent's capability model, auditable by what is and isn't a tool
5. **Extend via protocols and events, not inheritance** — Hooks (shell), Skills (markdown), MCP (RPC), Plugins (bundles)
6. **Recursive delegation is a design primitive** — Agents spawn agents; every tool interface should consider sub-agent callers
7. **Permission model is a trust spectrum** — A continuous spectrum from conservative to autonomous; ceiling is operator-controlled
8. **Modes are structural constraints with deliverables** — Plan and worktree modes are tool-driven state transitions with concrete completion artifacts
9. **Background work is a first-class primitive** — Tasks are persistent state, not transient processes; survive restarts; observable lifecycles
10. **Every interesting moment is structured** — Permission decisions, cost transitions, hook executions, compactions — all carry structured reasons
11. **The agent should know its own state** — Cost trajectory, budget headroom, context pressure, recent failures should be visible to the model

---

## When to Use

| Scenario                  | Example                                         |
| ------------------------- | ----------------------------------------------- |
| Design agent from scratch | "I want to build an agent that auto-fixes bugs" |
| Implementation questions  | "How should I design tool permission checks?"   |
| Review existing agent     | "Help me review this agent's architecture"      |
| Technology decisions      | "Should multi-agent use fork or fresh context?" |
| Performance optimization  | "My agent is consuming too many tokens"         |
| Context management        | "What compaction strategy should I pick?"       |
| Background tasks          | "How should I manage background task lifecycle?" |
| Autonomous runs           | "How do I control cost for an autonomous agent?" |

---

## Structure

```
claude-code-methodology/
├── SKILL.md                          # Main file: pattern recognition + workflow + worked example
├── README.md                         # Chinese version (primary)
├── README_EN.md                      # This file
└── references/
    ├── 01-tool-design.md             # Tool design patterns
    ├── 02-system-prompt.md           # System prompt architecture patterns
    ├── 03-permission-safety.md       # Permission & safety patterns
    ├── 04-multi-agent.md             # Multi-agent orchestration patterns
    ├── 05-token-economy.md           # Context & token economy patterns
    ├── 06-memory-state.md            # Memory & state management patterns
    ├── 07-extensibility.md           # Extensibility patterns
    ├── 08-conversation-loop.md       # Conversation loop & query engine patterns
    ├── 09-workflow-modes.md          # Workflow modes patterns
    ├── 10-task-lifecycle.md          # Task lifecycle patterns
    └── 11-observability.md           # Observability & cost governance patterns
```

---

## Language Agnostic

All patterns are **language-agnostic**. Reference docs use TypeScript and Python for examples, but the design principles apply to any stack: Python, Go, Rust, Java, LangChain, CrewAI, custom frameworks, etc.

---

## License

MIT
