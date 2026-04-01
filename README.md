<div align="center">

# Claude Code Methodology

> _"用生产级 Agent 的方式，来构建你的 Agent。"_

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-blueviolet)](https://claude.ai/code)
[![AgentSkills](https://img.shields.io/badge/AgentSkills-Standard-green)](https://agentskills.io)

<br>
每一个工具调用、每一条权限检查、每一次多 Agent 委托<br>
都是经过大规模生产验证的设计决策。<br>

**将这些决策提炼成 7 个维度，直接迁移到你的 Agent 项目。**

<br>

从工具设计到 Token 经济，从权限模型到多 Agent 编排<br>
覆盖 Agent 开发的完整生命周期

[安装](#安装) · [怎么使用](#怎么使用) · [七个维度](#七个维度) · [适用场景](#适用场景) · [**English**](README_EN.md)

</div>

---

## 这是什么

这是一个从 Claude Code 中提炼出的 **Agent 开发方法论**，打包成 Claude Code Skill。

所有模式和反模式均来自对 Claude Code 生产源码的逐行阅读，不是理论推演——每一条都经过了大规模生产环境的验证。

---

## 安装

```bash
npx skills add Dearest/claude-code-methodology
```

安装完成后，在任何 Claude Code 对话中涉及 Agent 开发时会自动触发。

---

## 怎么使用

安装后，直接在 Claude Code 中描述你的 Agent 开发需求即可触发：

- **"帮我设计一个能自动修 bug 的 Agent"** → 架构设计模式
- **"review 一下这个 Agent 的工具设计"** → Agent 审查模式
- **"我的 Agent token 消耗太大了"** → Token 经济指导
- **"multi-agent 怎么做任务委托？"** → 多 Agent 编排指导

### 三种工作模式

**1. 架构设计** — 从零设计新 Agent，逐步梳理需求，按维度分析，输出完整架构文档

**2. 实现指导** — 针对具体问题，定位对应维度，给出 Claude Code 的做法和背后原因，并适配到你的技术栈

**3. Agent 审查** — 对已有 Agent 代码或设计进行结构化审查，按维度评估，分级输出问题和建议

---

## 七个维度

| #   | 维度                               | 核心问题                               |
| --- | ---------------------------------- | -------------------------------------- |
| 1   | **工具设计** Tool Design           | 工具是否安全、描述清晰、可组合？       |
| 2   | **System Prompt 架构**             | 提示词是否可维护、高效、分层？         |
| 3   | **权限与安全** Permission & Safety | Agent 是否 fail-closed？能否被信任？   |
| 4   | **多 Agent 编排** Orchestration    | 委托、隔离、并行是否合理？             |
| 5   | **Token 经济** Token Economy       | 上下文在整个生命周期中是否被高效利用？ |
| 6   | **记忆与状态** Memory & State      | 知识如何在轮次和会话之间持久化？       |
| 7   | **可扩展性** Extensibility         | 能否在不修改核心逻辑的情况下扩展？     |

每个维度都有独立的参考文档，包含模式、反模式和设计检查清单。

---

## 核心原则

1. **Fail-closed 优于 fail-open** — 权限不明确时，默认拒绝
2. **Token 经济驱动架构** — 每一个结构决策都应考虑 token 成本
3. **静态与动态内容分离** — 稳定的指令可缓存，会话数据不缓存
4. **工具定义能力边界** — 工具集就是 Agent 的能力模型
5. **通过协议扩展，而非继承** — MCP、Hooks、Skills，而非子类化
6. **递归委托是设计原语** — Agent 可以产生 Agent
7. **权限模型是信任频谱** — 从保守到自治的连续光谱，而非开关

---

## 适用场景

| 场景           | 示例                                  |
| -------------- | ------------------------------------- |
| 从零设计 Agent | "我要做一个能自动修 bug 的 Agent"     |
| 实现细节咨询   | "tool 的权限检查应该怎么设计？"       |
| 审查现有 Agent | "帮我 review 这个 Agent 的架构"       |
| 技术选型参考   | "多 Agent 应该用 fork 还是 fresh？"   |
| 性能优化       | "我的 Agent token 消耗太大了怎么办？" |

---

## 目录结构

```
agent-methodology/
├── SKILL.md                          # 主文件：模式识别 + 工作流程
├── README.md                         # 本文件（中文）
├── README_EN.md                      # English version
└── references/
    ├── 01-tool-design.md             # 工具设计模式
    ├── 02-system-prompt.md           # System Prompt 架构模式
    ├── 03-permission-safety.md       # 权限与安全系统模式
    ├── 04-multi-agent.md             # 多 Agent 编排模式
    ├── 05-token-economy.md           # 上下文与 Token 经济模式
    ├── 06-memory-state.md            # 记忆与状态管理模式
    └── 07-extensibility.md           # 可扩展性模式
```

---

## 语言无关

方法论中的所有模式都是**语言无关的**。参考文档使用 TypeScript（源自 Claude Code）和 Python 作为示例，但设计原则适用于任何技术栈：Python、Go、Rust、Java、LangChain、CrewAI、自研框架等。

---

## License

MIT
