<div align="center">

# Claude Code Methodology

> _"用生产级 Agent 的方式，来构建你的 Agent。"_

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-blueviolet)](https://claude.ai/code)
[![AgentSkills](https://img.shields.io/badge/AgentSkills-Standard-green)](https://agentskills.io)

<br>
每一个工具调用、每一条权限检查、每一次多 Agent 委托、每一次上下文压缩——<br>
都是经过大规模生产验证的设计决策。<br>

**将这些决策提炼成 11 个维度，可直接迁移到任何 Agent 项目。**

<br>

从工具设计到 Token 经济，从权限模型到多 Agent 编排，<br>
从会话主循环到任务生命周期，从工作流模式到可观测性<br>
覆盖 Agent 开发的完整生命周期

[安装](#安装) · [怎么使用](#怎么使用) · [十一个维度](#十一个维度) · [适用场景](#适用场景) · [**English**](README_EN.md)

</div>

---

## 这是什么

一份从生产级 Agent 中提炼出的 **Agent 开发方法论**，打包成 Claude Code Skill。

每条模式与反模式都来自对一款规模化运行的真实 Agent 源码的逐层剖析——不是理论推演，而是大规模生产环境验证过的设计决策。所有模式与具体框架无关，可直接迁移到 Python / Go / Rust / LangChain / CrewAI / 自研框架。

---

## 安装

```bash
npx skills add Dearest/claude-code-methodology
```

安装完成后，在任何 Claude Code 对话中涉及 Agent 开发时会自动触发。

---

## 怎么使用

安装后，直接在 Claude Code 中描述你的 Agent 开发需求：

- **"帮我设计一个能自动修 bug 的 Agent"** → 架构设计模式
- **"review 一下这个 Agent 的工具设计"** → Agent 审查模式
- **"我的 Agent token 消耗太大了"** → Token 经济指导
- **"multi-agent 怎么做任务委托？"** → 多 Agent 编排指导
- **"我的 Agent 上下文经常爆掉，怎么做 compaction？"** → 会话主循环 + Token 经济
- **"后台任务怎么管理？"** → 任务生命周期
- **"我要做一个自主运行的 Agent，怎么控制成本？"** → 可观测性

### 三种工作模式

**1. 架构设计** — 从零设计新 Agent，逐步梳理需求，按维度分析，输出完整架构文档

**2. 实现指导** — 针对具体问题，定位对应维度，给出经过验证的做法和背后原因，并适配到你的技术栈

**3. Agent 审查** — 对已有 Agent 代码或设计进行结构化审查，按维度评估，分级输出问题与建议

---

## 十一个维度

| #   | 维度                                | 核心问题 |
| --- | ----------------------------------- | -------- |
| 1   | **工具设计** Tool Design            | 工具是否安全、自描述、可组合？ |
| 2   | **System Prompt 架构**              | 提示词是否模块化、可缓存？ |
| 3   | **权限与安全** Permission & Safety  | Agent 是否 fail-closed？决策是否结构化？是否有 operator 不可越过的上限？ |
| 4   | **多 Agent 编排** Orchestration     | 是否针对任务选择合适的拓扑（fork/fresh/worker/team/remote/scheduled）？ |
| 5   | **Token 经济** Token Economy        | 4 个 token 池是否各自采用合适的策略（缓存、按需加载、压缩、淘汰）？ |
| 6   | **记忆与状态** Memory & State       | 5 种状态分类是否清晰？持久化记忆是否文件化且自动触发？ |
| 7   | **可扩展性** Extensibility          | Hooks、Skills、MCP、Plugins 是否能在不改核心代码的前提下扩展？ |
| 8   | **会话主循环** Conversation Loop    | 主循环是否支持 streaming、中断、重试、保证不变量？ |
| 9   | **工作流模式** Workflow Modes       | 是否有结构化的模式原语（Plan 模式、Worktree 模式）控制阶段约束？ |
| 10  | **任务生命周期** Task Lifecycle     | 后台任务是否一等对象（状态、输出、取消、恢复）？ |
| 11  | **可观测性与成本** Observability    | 每一个关键时刻是否结构化、可归因、可查询？ |

每个维度都有独立参考文档，包含模式、反模式、设计检查清单与迁移代码示例。

---

## 核心原则

这是 11 个维度跨维度提炼出的最高层不变量：

1. **Fail-closed 优于 fail-open** — 权限不明确时默认拒绝；分类器异常时拒绝；输入异常时询问
2. **Token 经济驱动架构** — 每一个结构决策都应考虑 token 成本
3. **静态与动态分离** — 稳定指令进可缓存段；会话数据进动态段
4. **工具定义能力边界** — 工具集就是 Agent 的能力模型，能力可审计
5. **通过协议与事件扩展，而非继承** — Hooks、Skills、MCP、Plugins
6. **递归委托是设计原语** — Agent 可以产生 Agent，每个工具接口都要考虑被 sub-agent 调用的可能
7. **权限模型是信任频谱** — 从保守到自治的连续光谱；上限由 operator 控制
8. **模式是带交付物的结构化约束** — Plan 模式、Worktree 模式都是模型通过工具进入并必须产出可交付物的状态
9. **后台任务是一等原语** — 任务是持久化对象，不是临时进程；可跨重启恢复
10. **每一个关键时刻都是结构化的** — 权限决策、成本变化、Hook 执行、压缩——全部带结构化原因
11. **Agent 应该知道自己的状态** — 成本走势、预算余量、上下文压力、近期失败——这些都应该进到模型的上下文里

---

## 适用场景

| 场景           | 示例                                  |
| -------------- | ------------------------------------- |
| 从零设计 Agent | "我要做一个能自动修 bug 的 Agent"     |
| 实现细节咨询   | "tool 的权限检查应该怎么设计？"       |
| 审查现有 Agent | "帮我 review 这个 Agent 的架构"       |
| 技术选型参考   | "多 Agent 应该用 fork 还是 fresh？"   |
| 性能优化       | "我的 Agent token 消耗太大了怎么办？" |
| Context 管理   | "compaction 策略应该怎么选？"         |
| 后台任务       | "background task 应该怎么管理生命周期？" |
| 自主运行       | "Agent 自主运行怎么控制成本？"        |

---

## 目录结构

```
claude-code-methodology/
├── SKILL.md                          # 主文件：模式识别 + 工作流程 + 贯穿示例
├── README.md                         # 本文件（中文）
├── README_EN.md                      # English version
└── references/
    ├── 01-tool-design.md             # 工具设计模式
    ├── 02-system-prompt.md           # System Prompt 架构模式
    ├── 03-permission-safety.md       # 权限与安全模式
    ├── 04-multi-agent.md             # 多 Agent 编排模式
    ├── 05-token-economy.md           # 上下文与 Token 经济模式
    ├── 06-memory-state.md            # 记忆与状态管理模式
    ├── 07-extensibility.md           # 可扩展性模式
    ├── 08-conversation-loop.md       # 会话主循环模式
    ├── 09-workflow-modes.md          # 工作流模式模式
    ├── 10-task-lifecycle.md          # 任务生命周期模式
    └── 11-observability.md           # 可观测性与成本治理模式
```

---

## 语言无关

方法论中的所有模式都是**语言无关的**。参考文档使用 TypeScript 和 Python 作为示例，但设计原则适用于任何技术栈：Python、Go、Rust、Java、LangChain、CrewAI、自研框架等。

---

## License

MIT
