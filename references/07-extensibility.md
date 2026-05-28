# Extensibility Patterns

---

## The Core Insight

A production agent must be designable by people who don't (and shouldn't) touch its core. The mechanism is **extension through protocols and events, not through inheritance or code modification**.

Four orthogonal extension surfaces exist in a mature system:

| Surface | What it adds | Authored by | Activation |
|---|---|---|---|
| **Hooks** | Custom behavior at lifecycle events | Power user, in shell or script | Settings file declaration |
| **Skills** | Workflows triggered by intent | Anyone, in markdown | Auto-trigger on description match |
| **MCP servers** | New tools | Tool developer, separate process | Per-server config |
| **Plugins** | Bundled tools + skills + commands + hooks | Plugin author | Plugin install/uninstall |

The key is that each surface targets a different audience and a different kind of extension. A shell-script person writes hooks. A workflow-defining user writes skills. A tool developer writes MCP servers. A community contributor bundles their work as a plugin. None of them need to know the other layers.

---

## Pattern 1: Hooks — Lifecycle Events as Shell Commands

A hook is a **shell command the framework runs at a defined moment**, reading its stdin (event data as JSON), inspecting its exit code and stdout, and responding accordingly.

This design has three deliberate properties:

1. **Language-agnostic.** Anything that can run a command can be a hook: bash, Python, Node, Go, a compiled binary. Users don't have to learn the agent's internal language.

2. **Sandboxed by OS.** The hook runs in its own process with whatever permissions the user has, not the agent's internals. A misbehaving hook cannot corrupt the agent's state.

3. **Composable.** Multiple hooks can register for the same event; they run in order.

**The complete hook event taxonomy:**

| Event | When it fires | Can block? |
|---|---|---|
| `Setup` | Once, at agent install or first run | No |
| `SessionStart` | At the beginning of every conversation | Yes (abort session) |
| `UserPromptSubmit` | When the user sends a message, before the model sees it | Yes (modify or reject) |
| `PreToolUse` | Before any tool call, after permission decision | Yes (veto the call) |
| `PostToolUse` | After successful tool call | No (observe only) |
| `PostToolUseFailure` | After failed tool call | No (observe only) |
| `PermissionRequest` | When the framework would show a confirmation dialog | Yes (auto-decide) |
| `PermissionDenied` | When a tool call is blocked | No |
| `PreCompact` | Before the framework triggers compaction | Yes (defer or veto) |
| `PostCompact` | After compaction completes | No |
| `SubagentStart` | When a sub-agent is spawned | Yes (block spawn) |
| `SubagentStop` | When a sub-agent finishes | No |
| `Stop` | When the agent's main loop stops | No |
| `StopFailure` | When the agent stops due to error | No |
| `Notification` | When the framework would send an OS-level notification | Yes (suppress) |
| `TeammateIdle` | When an in-process teammate has been idle | No |
| `TaskCreated`, `TaskCompleted` | Background task lifecycle | No |
| `Elicitation`, `ElicitationResult` | URL-elicitation flows (MCP) | No |
| `ConfigChange` | Settings file modified mid-session | No (re-read state) |
| `WorktreeCreate`, `WorktreeRemove` | Worktree lifecycle | No |
| `InstructionsLoaded` | When CLAUDE.md or equivalent is loaded | No |
| `CwdChanged` | Working directory change within session | No |
| `FileChanged` | File system watch fires | No |

**Each event has a defined input contract** — the JSON shape sent on stdin. Hook authors write to that contract.

**Hooks that block return a special exit code** or structured output indicating the block. The framework reads the exit code and decides whether to proceed. A non-zero exit on a blocking hook means "veto, here is the message"; a zero exit means "proceed."

---

## Pattern 2: Hook Event Streaming and Always-Emitted Events

Even non-blocking hooks need observability. The framework emits each hook's lifecycle as events:

```typescript
type HookExecutionEvent =
  | { type: 'started',  hookId, hookName, hookEvent }
  | { type: 'progress', hookId, hookName, hookEvent, stdout, stderr, output }
  | { type: 'response', hookId, hookName, hookEvent, exitCode, outcome }
```

These events are how the UI shows "hook X is running" to the user, how telemetry tracks hook latency and failure rates, and how SDK consumers integrate.

**Two emission policies:**

- **Always emitted** — a low-noise allowlist (`SessionStart`, `Setup`) emits unconditionally. These are backward-compatible events that have always been visible.
- **Opt-in** — the broader set of events emits only when `includeHookEvents` is set (typically for SDK consumers and remote mode).

This avoids flooding default consumers with low-value events while keeping the data available for the consumers who want it.

**Pending event buffering.** If a hook event fires before a handler is registered (early in startup), the event is buffered up to a cap (`MAX_PENDING_EVENTS = 100`) and replayed when the handler attaches. This prevents lost startup events without unbounded buffering.

---

## Pattern 3: Hook Configuration Layering

Hooks are configured in settings files, layered the same way as permission rules:

```
managedSettings (operator)       ← cannot be disabled by user
userSettings                     ← personal hooks across all projects
projectSettings                  ← project-wide hooks, in version control
localSettings                    ← machine-local override (not committed)
```

A hook configuration looks like:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "validate-bash.sh" }
        ]
      }
    ],
    "SessionStart": [
      { "hooks": [{ "type": "command", "command": "setup-env.sh" }] }
    ]
  }
}
```

The `matcher` lets a hook target specific tools; without it, the hook fires for all calls of the event type.

**Settings change detection.** A hook can edit settings (e.g., add a permission rule). The framework watches the settings file and emits `ConfigChange` so the rest of the system can re-read state without a restart.

---

## Pattern 4: Skills — User-Defined Workflows in Markdown

A skill is a **markdown file declaring a workflow that activates by intent match**, not by user command.

Anatomy of a skill:

```markdown
---
name: pr-review-workflow
description: Step-by-step PR review with checklist and verification.
  Triggers when user says "review this PR", "check PR #N", or asks for
  code review of a pull request.
---

# PR Review Workflow

When the user asks you to review a PR:

1. Fetch the PR description and diff (use `gh pr view N`)
2. Read each changed file in full, not just the diff
3. Check: tests added? edge cases? error handling? security?
4. ...
```

**The frontmatter `description` is the activation trigger.** The framework computes a similarity match between the user's message and each skill's description. When the score exceeds a threshold (or when the user explicitly types `/skill-name`), the skill activates.

**Activation means injecting the skill content into the system prompt for that turn.** The model sees the skill body as additional instructions for the current task. After the task completes, the skill exits the prompt (it does not persist).

---

## Pattern 5: Skill Invocation as a Tool

A skill is not invoked by string-matching alone — it is invoked by calling the `Skill` tool with the skill name:

```
Skill({ skill: "pr-review-workflow" })
```

This indirection has three benefits:

1. **The model decides to invoke.** Even if the user's intent matched a skill, the model considers whether the skill actually fits the request and invokes it deliberately. False positives in the matching don't auto-activate; they surface the skill as available.

2. **Multiple skills can be considered.** When the message could match several skills, the model picks based on richer context than the matcher could see.

3. **The activation is auditable.** A turn that invoked a skill is distinguishable from one that didn't, in the message stream.

**The Skill tool has restrictions:**

- It can only invoke skills already in the registry — not arbitrary names.
- It cannot invoke a skill that is already running (no recursion within a skill).
- It produces a tool result with the skill content; the model reads that as its next instruction.

---

## Pattern 6: Progressive Disclosure in Skills

Skill bodies can be long, with examples, edge cases, references. Loading the full body for every match would be expensive. The pattern: **the matcher sees only the metadata; the body loads only when the skill activates.**

In the listing the model sees at session start:

```
Available skills:
- pr-review-workflow: Step-by-step PR review with checklist...
- commit-message-format: Generates conventional commits...
- debugging-protocol: Systematic debugging approach...
```

When the model calls `Skill({ skill: "pr-review-workflow" })`, the framework reads the full markdown body and returns it.

**Skill resources.** A skill can bundle additional files in a `resources/` or `references/` subdirectory. These are not loaded automatically; the skill body can reference them and the model can `FileRead` them on demand.

```
skills/
└── pr-review-workflow/
    ├── SKILL.md
    └── references/
        ├── security-checklist.md
        ├── performance-checklist.md
        └── examples/
            ├── good-review.md
            └── bad-review.md
```

This achieves three-level progressive disclosure: metadata → body → references. Token cost matches usage depth.

---

## Pattern 7: MCP — Standard Protocol for External Tools

The Model Context Protocol (MCP) is the framework for adding tools without modifying the agent's code. An MCP server is a separate process that:

1. Speaks the MCP protocol over a standard transport (stdio, HTTP, WebSocket)
2. Declares its tools with standard schemas
3. Executes tool calls against its own implementation

The agent host connects to the server, fetches the tool list, and registers those tools into its tool pool. To the model, MCP tools look indistinguishable from built-in tools.

**Why a protocol instead of a plugin API?**

- **Versionability.** The protocol has a version; servers and hosts can negotiate.
- **Language independence.** MCP servers in any language; the host doesn't care.
- **Sandboxing.** The server runs in its own process; a crash doesn't take down the agent.
- **Distribution.** A user installs an MCP server like they install any other CLI tool. No reaching into the agent's plugins directory.

**MCP server lifecycle:**

- The host launches the server on session start (or on demand)
- The host sends tool listing and resource discovery requests
- The host forwards each tool call from the model to the server
- The host handles the server's progress updates and final results
- The host shuts down the server on session end

**MCP tools surface with a prefix.** A tool from server `foo` named `bar` becomes `foo__bar` in the model's tool list. This namespaces tool names so two servers can both have a `search` tool without colliding.

---

## Pattern 8: MCP Server Connection Lifecycle and Caching

MCP servers can connect and disconnect during a session. This is operationally useful (a server crashes and restarts; the user installs a new server mid-conversation) but breaks the prompt cache.

**The mitigation:**

- The list of MCP-derived tools is included in the request body as part of the tool array. When the list changes, the cache misses.
- The framework places MCP instructions in an **uncached section** (the `DANGEROUS_uncachedSystemPromptSection` mentioned in `02-system-prompt.md`), with the documented reason "MCP servers connect/disconnect between turns."
- For known-from-start MCP servers, the framework can include their tool definitions in the initial cache-friendly schema; for late-connecting servers, the cache miss is unavoidable.

**Recommendation:** users with critical MCP servers should configure them to launch at session start, not lazily. This pays a startup cost once and gets cache benefits across the session, rather than paying a cache-miss cost on first use.

---

## Pattern 9: Plugins — Bundled Distribution

A plugin packages multiple extensions into a single installable unit:

```
my-plugin/
├── plugin.json              ← manifest
├── tools/                   ← MCP servers or framework tools
│   └── ...
├── skills/                  ← skill markdown files
│   ├── workflow-a/
│   └── workflow-b/
├── commands/                ← slash commands
│   └── ...
├── hooks/                   ← pre-built hook scripts
│   ├── pre-commit.sh
│   └── ...
└── settings.json            ← default settings the plugin contributes
```

The manifest declares:

- Plugin name, version, author
- Required platform / agent version
- Tool, skill, command, hook contributions
- Settings to add (with conflict resolution)

**Plugin installation does not modify the agent.** The plugin's contributions register into the relevant registries (tool registry, skill registry, command registry, hook registry). Uninstall reverses the registration.

**Plugins are versioned.** Multiple plugins may declare overlapping tools or skills; the conflict-resolution rule typically prefers later installations, with explicit override or namespacing for advanced cases.

**Plugins can declare other plugins as dependencies.** This is what lets community marketplaces work: one plugin pulls in a stack of dependencies, the user installs one thing, the framework resolves the tree.

---

## Pattern 10: Slash Commands as Code-Backed Skills

A slash command is a special skill: it has a name (the slash), an optional argument schema, and a body that executes either as instructions (like a skill) or as code (like a tool).

```markdown
---
name: commit
description: Generate a conventional commit message and create the commit.
allowed-tools: ["Bash", "FileRead"]
---

# /commit

Analyze the staged changes, draft a conventional commit message, and create the commit.

1. Run `git diff --staged` to see what's being committed
2. Categorize the changes (feat/fix/refactor/...)
3. Draft a message: `<type>(<scope>): <subject>`
4. Run `git commit -m "<message>"`
```

The user types `/commit` (with optional arguments after); the framework finds the `commit` slash command, expands its body into the next user message, and the model executes the instructions.

**Slash commands restrict tools.** A command can declare which tools it needs (`allowed-tools`), narrowing the tool surface for that command's execution. This makes commands more predictable and safer.

**Slash commands can take arguments.** The user types `/review #123`; the body sees `123` as `$1`. Argument schemas can be declared in the frontmatter.

---

## Pattern 11: Settings Layers and the Operator Ceiling

The configuration system is the cross-cutting backbone of every other extensibility surface — permission rules, hook configurations, plugin enable/disable, skill availability all flow through it.

The layered hierarchy:

```
1. managedSettings (operator/enterprise, deployed via MDM or org config)
2. policySettings
3. userSettings (~/.agent/settings.json — global per user)
4. projectSettings ({project-root}/.agent/settings.json — version controlled)
5. localSettings ({project-root}/.agent/settings.local.json — machine local)
6. flagSettings (built-in feature flag defaults)
7. cliArg (--flag overrides for this session)
```

Lower layers add and refine; higher layers (1, 2) are **bypass-immune** — even users with bypass-mode autonomy cannot override managed and policy settings.

**Each setting carries source attribution.** When the agent loads a rule from `userSettings`, it knows the rule's origin. This lets the UI explain "this auto-approval came from `userSettings` rule X" and lets audit answer "which settings layer contributed to this decision?"

---

## Pattern 12: Settings Sync and Managed Remote

For team and enterprise deployments, settings need to come from a central source, not be authored ad-hoc on each user's machine.

The pattern: **managed remote settings** — the framework periodically fetches updates from an operator-controlled endpoint and treats them as the `managedSettings` layer. This is how an org pushes a new deny-rule to every contributor's agent without each user having to install anything.

Key properties:

- **Authenticated fetch.** The agent uses operator-issued credentials.
- **Signed payloads.** The operator signs the settings; the agent verifies before applying.
- **Atomic apply.** A bad managed-settings push doesn't half-apply; either the whole new state activates or the previous state remains.
- **Audit trail.** Every managed-settings refresh logs what changed and why.
- **Soft-fail.** If the fetch fails, the agent continues with the last-known-good managed settings; it doesn't refuse to run.

This is the layer where bypass-immune operations are declared (see `03-permission-safety.md`, Pattern 9). Operator policy lives here.

---

## Extensibility Design Checklist

**Hooks**
- [ ] Is there a defined taxonomy of hook events with clear "what fires when" semantics?
- [ ] Can hooks block at the events where blocking makes sense (pre-tool, pre-compact, session-start)?
- [ ] Are hooks shell commands (language-agnostic) rather than in-process callbacks?
- [ ] Is hook execution observable via event streams (started, progress, response)?
- [ ] Is there a low-noise allowlist of always-emitted events vs an opt-in firehose?
- [ ] Do hook configurations live in the same layered settings hierarchy as everything else?

**Skills**
- [ ] Are skills declarative markdown with frontmatter, not code?
- [ ] Is the description field optimized for the matcher (triggers, when-to-use, examples)?
- [ ] Does skill activation go through a tool call (auditable, model-decided)?
- [ ] Is the matcher seeing metadata only, with the body loading on activation?
- [ ] Do skills support resources/references that load only on demand?
- [ ] Are user-invocable skills exposed as slash commands when needed?

**MCP**
- [ ] Is MCP the path for external tools, not a custom plugin API?
- [ ] Are MCP servers separate processes with stdio/HTTP/WebSocket transport?
- [ ] Are MCP tool names namespaced with a server prefix to avoid collision?
- [ ] Are MCP instructions in an uncached prompt section?
- [ ] Does the host gracefully handle MCP server crashes (restart, fail closed)?

**Plugins**
- [ ] Is there a plugin manifest format that declares versioned dependencies?
- [ ] Do plugins contribute via registration, not core modification?
- [ ] Are plugins uninstallable cleanly (reverse the registrations)?
- [ ] Is there a conflict-resolution rule when plugins overlap?

**Settings**
- [ ] Is there a clear precedence order of settings layers, documented?
- [ ] Are top layers (managed, policy) bypass-immune?
- [ ] Does every setting carry source attribution through the merge?
- [ ] Is there a way to push managed settings from a central source for team/enterprise?
- [ ] Are managed-settings updates authenticated, signed, atomic, and soft-failing?

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Plugin API requires understanding agent internals | Limits who can extend | Protocol-based extension (MCP) + declarative skills + shell hooks |
| Hooks as in-process callbacks in the agent's language | Excludes non-native authors; sandboxing is the agent's burden | Shell commands; OS-process sandboxing |
| Hook events are an undocumented growing set | Hook authors can't write reliable hooks | Explicit taxonomy with contracts |
| Hooks emit events to one consumer | UI, telemetry, SDK each rebuild the firehose | Single event stream with multiple handlers |
| All hook events always emitted | Floods default consumers with low-value events | Always-emitted allowlist + opt-in firehose |
| Skills as code in some DSL | Limits authoring audience | Markdown with frontmatter; ordinary writing |
| Skill matching loads full bodies | Token cost grows with skill catalog | Progressive disclosure: metadata → body → references |
| Skill auto-activates without model decision | False matches contaminate the prompt | Skill tool indirection; model decides |
| Custom plugin API instead of MCP | Reinvents transport, schema, lifecycle | Adopt MCP |
| MCP servers expected to run forever, no restart | Server crash kills tool surface for the session | Host restarts MCP servers, soft-fails on persistent failure |
| MCP instructions in cached static prefix | Late MCP connect busts cache forever | Uncached section with reason |
| Plugins overwrite agent code | Updates clobber plugin changes | Plugins register into registries, not the codebase |
| Settings stored in a single layer | Cannot separate operator, user, project | Layered hierarchy with attribution |
| Managed settings without signing | Vector for tampering | Signed payloads; verify before apply |
| Hooks block forever silently | Agent hangs without explanation | Per-hook timeouts; visible status in UI |
| Slash commands without scope restrictions | Commands inherit the full tool surface, including dangerous ones | `allowed-tools` per command |
