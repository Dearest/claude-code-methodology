# Workflow Modes Patterns

---

## The Core Insight

A workflow mode is a **structural restriction the model enters and exits deliberately, via dedicated tools, that changes what the agent can do for a span of work**.

Modes are not permissions. Permissions answer "is this specific call allowed?" Modes answer "what kind of work am I doing right now, and what should be off-limits during it?" Modes are also not configuration — they're not set once at startup; they change mid-conversation as the work changes.

The mode pattern shows up in three production cases:

| Mode | Allows | Forbids | Entry | Exit |
|---|---|---|---|---|
| **Plan mode** | Reads, search, exploration, asking | All writes, all destructive operations | `EnterPlanMode` tool | `ExitPlanMode` tool (with plan file) |
| **Worktree mode** | Writes within the worktree | Writes outside the worktree | `EnterWorktree` tool | `ExitWorktree` tool (commit / discard) |
| **Brief mode** (and similar persona shifts) | Concise output | Verbose output, elaboration | Implicit on tool invocation | Implicit on next user message |

The unifying property: **the mode tool is not just a flag flip; the act of calling it announces the intent and the constraint to both the model and the user**. The transition is auditable, reviewable, and reversible.

---

## Pattern 1: Modes as Tool-Driven State Transitions

A mode is implemented as a pair of tools:

- An **entry tool** that the model calls when it decides the work fits the mode
- An **exit tool** that the model calls when the work is done or the constraint should lift

```python
# Pseudocode for a Plan Mode entry
def call_enter_plan_mode(input, context):
    context.set_app_state(lambda s: s.with_permission_mode(
        'plan',
        prePlanMode=s.permission_context.mode  # save current mode for restore
    ))
    return ToolResult(data="Entered plan mode. Reads allowed; writes blocked.")
```

Three properties make this pattern work:

1. **The transition is visible.** Both the model (in its tool call history) and the user (in the UI) see the mode change. Nothing hidden.

2. **The mode is part of the AppState.** Other parts of the system (the permission engine, the UI, the loop) read the current mode and adapt — the permission engine blocks writes when mode is `plan`, the UI shows a blue mode indicator, the loop respects the constraint.

3. **The exit tool can require completion artifacts.** `ExitPlanMode` insists that a plan file has been written. The model can't just exit; it has to produce the deliverable that the mode existed to create. This makes the mode goal-directed.

---

## Pattern 2: Why Modes Beat Per-Call Permission

The naive alternative to modes is "ask the user for each write." The result is alarm fatigue: the user mashes "approve" on every prompt because there are too many.

Modes invert this. Instead of "ask for each call," modes say:

> "For the next 30 minutes, all calls are reads. If the model tries to write, the framework refuses immediately — no prompt, no user decision needed."

The user gets one decision (when the mode is entered, optionally) and total assurance that the constraint holds. The model gets a stable contract — it doesn't have to second-guess every tool selection.

**Modes also constrain the model's planning.** A model in plan mode that internally considers "I should write to file X to test this" can reject the consideration without trying — the model knows the call would fail. Without mode, the model would try, get blocked, and have to recover. The mode makes the constraint a known input.

**Modes are honest about scope.** "Plan mode" cleanly demarcates "we're planning, not implementing" — a phase distinction that exists in every non-trivial project. Without the mode, the same boundary has to be expressed via prose ("don't write yet, just plan"), and the model has to interpret prose, which is unreliable.

---

## Pattern 3: Plan Mode — Read-Only Exploration With a Deliverable

The canonical mode. When the model decides a task is non-trivial enough to warrant an upfront design (which is most non-trivial tasks), it calls `EnterPlanMode`. The system:

- Records `prePlanMode` (the mode the user was in)
- Switches the active permission mode to `plan`
- Injects a plan-mode attachment into the next system reminder, instructing the model on the plan-mode workflow
- Shows the user a mode indicator

In plan mode, the model:

1. Reads files, searches the codebase, explores the structure
2. Designs an approach
3. Writes the plan to a plan file (path provided in the plan-mode system message)
4. Optionally uses the structured-question tool to clarify with the user
5. Calls `ExitPlanMode` with no arguments; the framework reads the plan from the file and presents it to the user for approval

The user reviews the plan and either approves, edits, or rejects. On approval, the mode exits, permissions revert to `prePlanMode`, and implementation begins with the plan as context.

**When to use plan mode**, per the canonical tool prompt:

- New feature implementation
- Multiple valid approaches that need user input
- Code modifications that affect existing behavior
- Architectural decisions
- Multi-file changes (more than 2–3 files)
- Unclear requirements that need exploration

**When NOT to use plan mode:**

- Trivial single-file edits
- Pure research tasks (the deliverable isn't a plan, it's findings)
- Tasks the user explicitly described step-by-step

---

## Pattern 4: prePlanMode and Mode Restoration

When the model enters plan mode mid-session, it might have been in `acceptEdits`, `bypassPermissions`, or some other non-default mode. Exiting plan mode should restore that, not drop the user back to `default`.

The pattern: **record the prior mode at entry; restore it at exit.**

```typescript
type ToolPermissionContext = {
  mode: PermissionMode
  prePlanMode?: PermissionMode  // set on plan-mode entry; cleared on exit
  // ...
}
```

The `prePlanMode` field is part of the permission context, so it travels with the rest of permission state. On exit:

```python
def exit_plan_mode(context):
    prior = context.permission_context.pre_plan_mode or 'default'
    context.set_app_state(lambda s: s.with_permission_mode(
        prior,
        prePlanMode=None  # clear the marker
    ))
```

**Why this matters operationally.** Power users run in `acceptEdits` to skip per-edit prompts. They enter plan mode for a design discussion. Without restoration, they'd have to manually re-enter `acceptEdits` after every plan mode exit. With restoration, plan mode is a transient phase that doesn't disrupt their normal workflow.

---

## Pattern 5: Worktree Mode — Filesystem Isolation as a Mode

For speculative changes (refactors, large rewrites, "let me try X and see"), the right pattern is to run in a separate git worktree. The mode tool pair: `EnterWorktree` and `ExitWorktree`.

Entry:

1. Create a new git worktree at a temp path, on a new branch
2. Change the agent's working directory to the worktree
3. Inject a system message: "You're now in worktree X on branch Y. Changes here are isolated from the main tree."
4. Set up the abort hook so an interrupted worktree session can be torn down cleanly

Exit:

1. Inspect the worktree's diff
2. Either: commit the changes, propose them as a PR, or discard them
3. Tear down the worktree
4. Restore the agent's cwd to the main tree

**Two exit shapes:**

- `ExitWorktree(action='keep')` — leave the worktree, the branch, the commits in place. The user can review later.
- `ExitWorktree(action='discard')` — delete the worktree and the branch. Clean wipe.

**Why "mode" is the right abstraction.** The set of valid operations changes when the worktree is active: reading files now means reading from the worktree, writing means writing to the worktree, `git status` shows the branch, `git diff` shows the branch's changes. None of these need to be parameterized with "which directory?" — the mode supplies the answer.

Worktrees specifically are also covered in `04-multi-agent.md` (Pattern 6) as a sub-agent isolation mechanism. Worktree mode in the parent agent is the same primitive applied to the parent's own work.

---

## Pattern 6: The Mode-Entry Tool's Prompt Teaches the Discipline

The model needs to know WHEN to enter a mode. The prompt of the entry tool is the teaching surface:

```
Use EnterPlanMode proactively for any non-trivial implementation task. Getting user
sign-off on your approach before writing code prevents wasted effort.

## When to Use
1. New Feature Implementation: "Add a logout button" - where should it go?
2. Multiple Valid Approaches: "Add caching" - Redis vs in-memory vs file?
3. Code Modifications: "Update the login flow" - what exactly should change?
4. Architectural Decisions: "Add real-time updates" - WebSockets vs SSE vs polling?
5. Multi-File Changes: "Refactor the auth system"
6. Unclear Requirements: "Make the app faster" - need to profile first

## When NOT to Use
1. Simple single-file edits where the approach is obvious
2. Pure exploration (use Read/Grep directly without mode)
3. The user already gave you a step-by-step plan
```

**The prompt for the exit tool teaches the discipline differently.** It tells the model what the deliverable must look like, what to do BEFORE exiting, and what NOT to do:

```
Before exiting, ensure your plan is complete and unambiguous:
- If you have unresolved questions, ask via AskUserQuestion (in earlier phases)
- Once your plan is finalized, call this tool to request approval

Do NOT use AskUserQuestion to ask "Is this plan okay?" — that's what THIS tool does.
ExitPlanMode inherently requests approval.
```

The negative guidance is as important as the positive. Without "do NOT ask 'is this okay?'", the model would tend to ask before exiting, doubling the user's confirmation work.

---

## Pattern 7: Mode-Triggered System Messages

When a mode is entered, the framework can inject mode-specific instructions into the next system reminder. This is how "plan mode" turns into actual changed behavior — the system reminder tells the model:

```
<system-reminder>
You are now in plan mode.

Write your plan to: {plan_file_path}

The plan should include:
- The approach (one paragraph)
- The files you'll modify
- The new files you'll create
- The testing strategy
- Open questions, if any

When the plan is complete, call ExitPlanMode to request user approval.
</system-reminder>
```

This message is *dynamic* — it knows the plan file path, the current session, and any per-session customization. It activates only when the relevant mode is active.

**Mode reminders are time-bound.** They live for the duration of the mode. When the mode exits, the reminder stops being injected. This keeps the context lean — the instructions are present when they're relevant and gone when they're not.

---

## Pattern 8: Modes for Sub-Agents

A sub-agent can be spawned directly into a mode. For example, a "research worker" can be a sub-agent whose `agentDefinition` declares `permissionMode: 'plan'`. The worker starts in plan mode, structurally cannot write, and exits the mode (or just exits) when its research is complete.

**This composes naturally with the multi-agent topology.** A coordinator can launch:

- A research worker in plan mode (cannot accidentally mutate)
- An implementation worker in worktree mode (mutations isolated)
- A verification worker in plan mode (read-only inspection)

Each worker's mode is selected by the coordinator at spawn time. The mode is part of the agent definition or the spawn parameters, not the worker's runtime decision.

---

## Pattern 9: Mode Composition and Mode Transitions

Modes can compose. A common pattern: enter plan mode at the start of a task, exit plan mode after approval, enter worktree mode for implementation, exit worktree mode with commit.

The framework should permit reasonable transitions and forbid nonsense ones:

| From | To | Allowed? |
|---|---|---|
| `default` | `plan` | Yes (entering plan to design) |
| `plan` | `default` | Yes (exiting plan after approval) |
| `plan` | `worktree` | No (must exit plan first) |
| `default` | `worktree` | Yes (entering worktree for speculative writes) |
| `worktree` | `default` | Yes (exiting worktree) |
| `plan` | `plan` | No (already in plan, no-op or error) |

Forbidden transitions surface as tool-call errors with explanatory messages, not as silent no-ops. The model needs to know its attempt failed and why.

---

## Pattern 10: When to Invent a New Mode

Three questions determine whether a new mode is the right abstraction:

1. **Is there a span of work with a clear start and end?** If the constraint is per-call, it's a permission rule, not a mode. If it spans many turns of work, it might be a mode.

2. **Does the model benefit from knowing the constraint upfront?** Modes change the model's planning. A constraint that only matters once doesn't need to be a mode; one the model needs to internalize across many decisions does.

3. **Is there a meaningful deliverable at the end?** Modes that exit on completion (plan file written, worktree committed) are more useful than modes that just toggle a flag. The deliverable is what the mode produces.

If all three are yes, a mode is appropriate. If one is no, you probably want a different abstraction:

- Per-call constraint without a span → permission rule
- Span without a clear "I'm doing this kind of work" → background setting
- Span without a deliverable → maybe just a system reminder

**Resist mode proliferation.** Each new mode adds cognitive load for the model and complexity for the framework. Three core modes (plan, worktree, default) cover most real cases. A fourth or fifth should be justified by a concrete unsolvable problem in the existing modes.

---

## Workflow Modes Design Checklist

**Architecture**
- [ ] Is the mode part of `AppState`, accessible to other subsystems?
- [ ] Does the permission engine respect the mode's structural constraints?
- [ ] Does the UI surface a visible indicator when a non-default mode is active?

**Entry tool**
- [ ] Is there a clear `WhenToUse` taxonomy in the entry tool's prompt?
- [ ] Is there a clear `WhenNotToUse` section that prevents over-application?
- [ ] Does the entry tool record any prior state needed for restoration on exit?

**Exit tool**
- [ ] Does the exit tool require any completion artifacts (a plan file, a commit)?
- [ ] Does the exit tool restore the prior mode (e.g., `prePlanMode`)?
- [ ] Does the exit tool's prompt forbid redundant confirmations?

**Mode-triggered behavior**
- [ ] Are mode-specific system messages injected when the mode is active?
- [ ] Do those messages reference dynamic per-session paths and parameters?
- [ ] Are the messages cleared when the mode exits?

**Composition**
- [ ] Are mode transitions explicitly allowed or forbidden, with helpful errors?
- [ ] Can sub-agents be spawned into specific modes?
- [ ] Can modes compose naturally (sequential, not nested)?

**Restoration**
- [ ] Is prior mode state recorded at entry?
- [ ] Is it restored on exit?
- [ ] Is the restoration also called on session restore from a resumed conversation?

**When to invent**
- [ ] Does the proposed mode have a clear span, model benefit, and deliverable?
- [ ] Are the three core modes (default, plan, worktree) not sufficient?

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Mode as a config flag set at startup | Can't transition mid-session; doesn't reflect actual workflow phases | Mode as tool-driven AppState transition |
| Hidden mode change | User confused about why the agent's behavior changed | Mode-entry tools that announce the transition visibly |
| Mode without deliverable | Mode becomes a no-op the model exits without producing anything | Exit tool requires completion artifact |
| No prior-mode restoration | Power users in `acceptEdits` get bounced to `default` after every plan-exit | `prePlanMode` (or equivalent) stored and restored |
| Permission rules trying to express mode | Many per-call confirmations; alarm fatigue | One mode transition; structural constraint for the span |
| Mode entry without `WhenNotToUse` | Mode applied to trivial tasks; user annoyed | Explicit negative guidance in prompt |
| Mode entry tool that also takes the work as input | Tool overloaded; one tool to enter, separate tools to do the work | Separate entry from execution |
| Exit tool that asks "is this okay?" | Redundant with the structural request-approval intent | Forbid the redundant question in the prompt |
| Mode reminders inject permanently | Context bloat with stale instructions | Mode reminders are scope-bound to mode duration |
| Forbidding all mode transitions | Workflow rigid; can't enter plan from worktree, etc. | Allow sensible transitions, forbid only nonsense |
| Mode proliferation | Cognitive load explodes; user can't remember what mode means what | Resist new modes; reuse defaults and permissions where possible |
| Sub-agent inherits parent's mode by accident | Worker spawns in plan mode and can't do its job | Sub-agent mode is declared in agent definition or spawn params, not inherited |
| Mode in pure markdown instructions ("be in plan mode now") | Model interprets prose unreliably; constraint sometimes ignored | Real structural mode with tool-enforced constraints |
