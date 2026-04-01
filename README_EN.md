<div align="center">

# Agent Methodology

> _"Build your agents the way production agents are built."_

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-blueviolet)](https://claude.ai/code)
[![AgentSkills](https://img.shields.io/badge/AgentSkills-Standard-green)](https://agentskills.io)

<br>
Every tool call, every permission check, every multi-agent delegation —<br>
is a design decision battle-tested at scale.<br>

**Distilled into 7 dimensions, ready to apply to your own agent projects.**

<br>

From tool design to token economy, from permission models to multi-agent orchestration<br>
covering the full lifecycle of agent development

[Install](#install) · [Usage](#usage) · [7 Dimensions](#7-dimensions) · [When to Use](#when-to-use) · [**中文**](README.md)

</div>

---

## What is this

An **Agent development methodology** extracted from Claude Code, packaged as a Claude Code Skill.

Every pattern and anti-pattern is derived from reading the actual production source code — not theory. Each one has been validated at scale in production environments.

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

### Three Modes

**1. Architecture Design** — Design a new agent from scratch. Walk through requirements, analyze by dimension, produce a full architecture document.

**2. Implementation Guidance** — For specific questions, map to the right dimension, explain Claude Code's approach and reasoning behind it, adapted to your tech stack.

**3. Agent Review** — Structured review of existing agent code or design, evaluated by dimension, with prioritized findings and recommendations.

---

## 7 Dimensions

| #   | Dimension                      | Core Question                                         |
| --- | ------------------------------ | ----------------------------------------------------- |
| 1   | **Tool Design**                | Are tools safe, clearly described, and composable?    |
| 2   | **System Prompt Architecture** | Is the prompt maintainable, efficient, and layered?   |
| 3   | **Permission & Safety**        | Does the agent fail-closed? Can it be trusted?        |
| 4   | **Multi-Agent Orchestration**  | Are delegation, isolation, and parallelism sound?     |
| 5   | **Token Economy**              | Is context utilized efficiently across the lifecycle? |
| 6   | **Memory & State**             | How does knowledge persist across turns and sessions? |
| 7   | **Extensibility**              | Can you extend without modifying core logic?          |

Each dimension has its own reference document with patterns, anti-patterns, and a design checklist.

---

## Core Principles

1. **Fail-closed over fail-open** — Default to deny when permissions are ambiguous
2. **Token economy drives architecture** — Every structural decision should account for token cost
3. **Separate static and dynamic content** — Stable instructions can be cached; session data cannot
4. **Tools define capability boundaries** — The toolset is the agent's capability model
5. **Extend via protocols, not inheritance** — MCP, Hooks, Skills — not subclassing
6. **Recursive delegation is a design primitive** — Agents can spawn agents
7. **Permission model is a trust spectrum** — A continuous spectrum from conservative to autonomous, not a toggle

---

## When to Use

| Scenario                  | Example                                         |
| ------------------------- | ----------------------------------------------- |
| Design agent from scratch | "I want to build an agent that auto-fixes bugs" |
| Implementation questions  | "How should I design tool permission checks?"   |
| Review existing agent     | "Help me review this agent's architecture"      |
| Technology decisions      | "Should multi-agent use fork or fresh context?" |
| Performance optimization  | "My agent is consuming too many tokens"         |

---

## Structure

```
agent-methodology/
├── SKILL.md                          # Main file: pattern recognition + workflow
├── README.md                         # Chinese version (primary)
├── README_EN.md                      # This file
└── references/
    ├── 01-tool-design.md             # Tool design patterns
    ├── 02-system-prompt.md           # System prompt architecture patterns
    ├── 03-permission-safety.md       # Permission & safety patterns
    ├── 04-multi-agent.md             # Multi-agent orchestration patterns
    ├── 05-token-economy.md           # Token economy patterns
    ├── 06-memory-state.md            # Memory & state management patterns
    └── 07-extensibility.md           # Extensibility patterns
```

---

## Language Agnostic

All patterns are **language-agnostic**. Reference docs use TypeScript (from Claude Code) and Python for examples, but the design principles apply to any stack: Python, Go, Rust, Java, LangChain, CrewAI, custom frameworks, etc.

---

## License

MIT
