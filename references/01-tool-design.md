# Tool Design Patterns

---

## The Core Insight

A tool is not a function. A tool is a **declarative specification object** that bundles five orthogonal concerns into a single contract:

1. **Capability** — what it does (the call function)
2. **Self-description** — separate descriptions for the model, the UI, and the user
3. **Safety profile** — read-only / destructive / concurrency-safe, declared per input
4. **Lifecycle behavior** — how it loads, when it interrupts, how it dedupes, what it returns
5. **Display contract** — what the user sees, what the model sees, what gets cached

This separation is what lets a production agent host 40+ tools without a giant orchestrator switch statement. The permission engine, the parallelism scheduler, the UI renderer, the token budgeter, and the cache controller all consume tool metadata through stable interfaces — none of them need to know what the tool actually does.

**The litmus test:** a new tool should be addable to your agent without modifying anything outside its own folder. If you have to touch the permission engine, the UI, or the loop to ship a tool, your tool interface is too thin.

---

## Pattern 1: Per-Input Safety Metadata

**Problem.** A tool's safety profile is not constant — it depends on the input. `git status` is read-only; `git reset --hard` is destructive. A blanket `isDestructive = true` on the Git tool would force confirmation on every command, including the safe ones, and users would learn to mash "yes". A blanket `false` would auto-approve disaster.

**The pattern.** Every safety property is a **function of the input**, not a static boolean.

```
tool.isReadOnly(input)        → boolean
tool.isDestructive(input)     → boolean
tool.isConcurrencySafe(input) → boolean
```

These three properties are the foundation of the entire permission system, the parallelism scheduler, and the UI affordances (confirm dialog, spinner color, undo button).

**Worked example.** A shell tool:

```python
class ShellTool:
    READ_ONLY_PREFIXES = {"ls", "pwd", "cat", "head", "tail", "git status", "git log", "git diff"}
    DESTRUCTIVE_PATTERNS = [r"rm -rf", r"git reset --hard", r"DROP TABLE", r":\(\)\{ :"]

    def is_read_only(self, input):
        cmd = input["command"].strip()
        return any(cmd.startswith(p) for p in self.READ_ONLY_PREFIXES)

    def is_destructive(self, input):
        cmd = input["command"]
        return any(re.search(p, cmd) for p in self.DESTRUCTIVE_PATTERNS)

    def is_concurrency_safe(self, input):
        # Read-only commands are safe to run in parallel; writes are not.
        return self.is_read_only(input)
```

**Principle.** Safety metadata is **self-declared by the tool, per input**. The orchestrator never inspects command strings; it asks the tool.

**Anti-pattern.** Safety logic in the orchestrator:

```python
# BAD: orchestrator knows too much, knowledge duplicated
if tool_name == "shell" and "rm -rf" in args["command"]:
    confirm()

# GOOD: every consumer trusts the tool
if tool.is_destructive(args):
    confirm()
```

---

## Pattern 2: Three Audiences, Three Descriptions

A tool has three text consumers. They need different things.

| Field | Audience | Length | Content |
|---|---|---|---|
| `prompt` (or `getModelDescription`) | The model | Long (200–2000 tokens) | What it does, WHEN to use, WHEN NOT to use, parameters with examples, failure modes, alternatives |
| `description` | The UI summary line | Very short | Imperative active phrase: "Run shell command" |
| `userFacingName` | The collapsed display name | One word, or empty string | "Bash" — or `""` to hide from UI entirely |

**The model-facing prompt is the most important text in your agent.** It is the only place the model learns to choose your tool over the other 39. Cheap prompts produce confused models that pick the wrong tool.

**Structure for the model-facing prompt:**

```
[Headline: one sentence on what it does]

## When to Use
- Bullet list of triggering scenarios

## When NOT to Use
- Bullet list of scenarios where a different tool is better
- Name the alternative tool by exact name

## Parameters
- param_name: meaning, value range, examples

## Output
- What the result looks like, error modes, how to recover

## Examples
<example>
<scenario>...</scenario>
<assistant_action>...</assistant_action>
<reasoning>...</reasoning>
</example>
```

The **WHEN NOT TO USE** section is the highest-leverage paragraph in a tool prompt. It is where you steer the model away from your tool toward better alternatives. Tools without a "do not use" section get over-selected.

**The empty `userFacingName`.** A tool can set `userFacingName = ""` to disable UI display entirely. This is the correct setting for tools whose only side effect is updating agent-internal state (e.g., a todo-list updater, a planning-mode toggle). They are mechanically necessary but should not clutter the user's transcript.

---

## Pattern 3: Permission Check As Four-Valued Decision

The permission interface on each tool returns one of four values:

| Result | Meaning | Caller behavior |
|---|---|---|
| `allow` | Auto-approve | Proceed to execution |
| `deny` | Auto-reject | Block with explanation |
| `ask` | Tool insists on confirmation | Show dialog regardless of mode |
| `passthrough` | "I have no opinion" | Defer to outer permission engine |

`passthrough` is the most important and the most under-used. It encodes humility: the tool says "I know my domain, but I do not know the user's preferences or the operator's policy — let the outer system decide." A tool that returns `allow` or `deny` for everything is overreaching.

**Permission check can also mutate the input.** A tool's `checkPermissions` returns not just a verdict but a possibly-modified input:

```python
def check_permissions(self, input, context):
    # Normalize a relative path before any approval; the user sees
    # and approves the absolute form.
    canonical = os.path.realpath(input["path"])
    if canonical.startswith("/home/" + context.user):
        return PermissionResult(
            behavior="allow",
            updated_input={**input, "path": canonical},
        )
    return PermissionResult(behavior="ask", updated_input={**input, "path": canonical})
```

This is how a tool enforces invariants (canonicalization, glob expansion, secret redaction) at the permission boundary, so that downstream code can trust its input.

**The classifier summary.** When the outer permission engine invokes an AI classifier, sending the full input is wasteful — large file contents, long command strings, base64 blobs. Tools should provide a `toAutoClassifierInput` function that returns a lossy summary:

```python
def to_auto_classifier_input(self, input):
    # Strip the file content, keep the path and intent
    return f"write to {input['path']} (size: {len(input['content'])} chars)"
```

This is consumed only by the classifier; the human-facing prompt and the actual execution still see the original input.

---

## Pattern 4: Deferred Loading + Discovery

**The token math.** Forty tools × ~500 tokens per schema = 20 000 tokens per request, billed on every turn, in every session, in every agent. At a million-turn-per-day fleet, that is dollars per agent per day.

**The pattern.** Tools split into two populations:

- **Always-loaded** — the ~8–12 tools used in 80%+ of turns: file read, glob, grep, shell, edit, the orchestrator's task tool. Always in the request body.
- **Deferred** — everything else. Not in the request by default. Discoverable through a built-in `tool_search` tool that the model can call when it senses it needs something it doesn't have.

A deferred tool declares a **`searchHint`** — a short matching phrase that the search tool ranks against the model's query.

**Format rules for `searchHint`:**

1. 3–10 words; no trailing punctuation
2. Use plain capability vocabulary ("manage browser windows", "compress images")
3. **Avoid words already in the tool name.** If the tool is `BrowserTool`, do not put "browser" in the hint — the name will match anyway. Use the hint for the *concept* the model would search for in its mental model. (For `NotebookEdit`, the hint should include "jupyter" — that's the word the model thinks in.)
4. Avoid jargon the model would not naturally produce

**The search tool's contract.** It returns the full schemas of the matched deferred tools, which become callable in the same turn or the next. From the model's perspective, the toolset expanded mid-conversation. The cost is one extra round-trip for the search; the savings is 20 000 tokens per turn.

**A deferred tool can still be `alwaysLoad: true` for specific agent types.** A search agent gets WebSearch always-loaded; a coding agent gets it deferred. This is per-agent, not per-tool.

---

## Pattern 5: Result Size Management

**The problem.** Tool outputs are the fastest-growing part of context in a real session. A single `grep -r` against a large repo, a single `ls -R` against `node_modules`, a single curl against a paginated API — any of these can blow the context window in one call.

**Per-tool result budget.** Every tool declares a `maxResultSizeChars`. When the output exceeds it, the tool chooses its own truncation strategy:

| Strategy | When to use | Example |
|---|---|---|
| **Tail-truncate** (keep the beginning) | Output is more informative early (errors at top, exit code last) | Shell commands |
| **Head-truncate** (keep the end) | Output is more informative late (test runs, "OK" at the end) | Test runners |
| **Middle-truncate** | Both ends matter (file with start and end visible, middle elided) | File preview |
| **Pagination** | The output is structured and the model can request more | File read (line ranges) |
| **Disk offload** | The output is too valuable to discard, model needs random access | Large search results, build artifacts |

The truncation strategy is the **tool author's choice**, not the framework's. The framework gives the tool the budget; the tool decides how to spend it.

**Disk offload pattern:**

```
if len(result) > MAX:
    path = persist_to_disk(result)
    return f"[Output too large ({len(result):,} chars). Saved to: {path}.\n"
           f"Use FileRead with offset/limit to inspect sections.]"
```

The model can then read the parts it cares about with another tool, instead of holding the whole result in context.

**Conversation-level result budget.** Independent of the per-tool budget, the agent enforces a **cumulative tool-result budget** for the whole conversation. When the total approaches its ceiling, the *oldest* tool results in history are replaced with `[Result evicted to save context]`. This is invisible to the model in the active turn but reclaims context for the future. The eviction is tracked per-thread so subagents don't poison their parent's history.

---

## Pattern 6: The Result Protocol

A tool result is not just "data". The result object exposes four channels into the agent runtime:

```typescript
type ToolResult<T> = {
  data: T                                  // The result for the model
  newMessages?: Message[]                  // Messages to inject into history
  contextModifier?: (ctx) => ctx           // Mutate the context (rare)
  mcpMeta?: { _meta, structuredContent }   // MCP protocol pass-through
}
```

**`data`** is the value the model sees, formatted by `mapToolResultToToolResultBlockParam`. Crucially, **the model-facing string is not the same as `data`** — `data` is the structured payload your post-processing code consumes; the model sees a serialized projection. This lets a tool record rich structured output for analytics and renderers while sending a concise summary to the model.

**`newMessages`** lets a tool *inject* messages into conversation history. Use cases:

- A long-running tool injects a system message "Output truncated at 50K chars, full output at /tmp/build.log"
- A subagent-spawning tool injects the subagent's final summary as an assistant message
- A file-edit tool injects a synthetic user message with the new file content (so the next read is from cache)

This is powerful and dangerous. Injected messages persist and bill on every subsequent turn. Use sparingly.

**`contextModifier`** lets a tool mutate the `ToolUseContext` for *subsequent* tool calls in the same turn. It is only honored for non-concurrency-safe tools (because two concurrent tools could each modify the same context, and the merge is ambiguous). Use cases: changing the working directory, expanding the allowlist after the user grants permission, registering a cleanup hook.

**`mcpMeta`** is a pass-through bag for MCP-tool consumers. Tools you author yourself usually leave this empty.

---

## Pattern 7: Cognitive Scaffolds — Tools With No External Effect

**The pattern.** Some tools do not call out to the world. They exist to **shape how the agent thinks**.

The classic examples:

- **Todo-write** — agent writes its plan as a structured checklist and updates status as it works
- **Ask-user-question** — agent asks a multiple-choice question with optional previews, instead of free-text
- **Enter-plan-mode / Exit-plan-mode** — agent declares "I am now planning, no writes"
- **Brief** — agent crystallizes its understanding before acting

These tools have no real side effect — they just store a structure in session state. But the *prompt of the tool* and the *act of calling it* shape the model's behavior:

- A model with `todo_write` available, used correctly, plans-then-executes. Without it, the model jumps to code.
- A model with `ask_user_question` asks structured questions. Without it, the model asks free-text and misses options.
- A model with `enter_plan_mode` self-restricts to reads. Without it, the model writes prematurely.

**The recipe.** When you find yourself repeatedly instructing the agent in prose ("first plan, then…", "ask me before…", "consider three options…"), promote that instruction to a tool. The tool's existence becomes the instruction.

**Cognitive scaffolds can carry post-completion nudges.** After a tool runs, the result-to-string mapping can include conditional reminders:

```python
def map_result(self, data, tool_use_id):
    base = "Todos updated."
    if data.all_done and not data.had_verification_step:
        nudge = "\nNOTE: You closed out the list without a verification step. " \
                "Before summarizing, run the verifier agent."
        return base + nudge
    return base
```

These nudges fire at the **exact moment** the model is about to skip a step. They are much cheaper and more reliable than restating the rule in the system prompt.

---

## Pattern 8: Interrupt, Concurrency, and Deduplication

**Interrupt behavior.** What should happen if the user sends a new message while a tool is mid-execution?

```
interruptBehavior: 'cancel' | 'block'
```

- **`cancel`** — abort the tool, discard its partial output, process the new message. Use for tools that are safe to abandon mid-flight (web fetch, search).
- **`block`** — finish the tool first, then process the new message. Use for tools that are unsafe to interrupt (a file write halfway done, a database transaction).

A tool implementing `cancel` must respect the `AbortController` passed in the context — long loops check `signal.aborted` periodically.

**Concurrency safety per input.** `isConcurrencySafe(input)` returns whether *this specific call* can run in parallel with sibling tool calls. Reading two different files is safe. Writing two different files is safe. Writing the same file twice is not. The tool inspects the input to decide.

```python
def is_concurrency_safe(self, input):
    # Multiple reads ok, multiple writes to same path not ok
    if input["mode"] == "read":
        return True
    return False  # writes are serialized
```

The parallelism scheduler reads this flag to decide which tool calls to run concurrently versus in sequence.

**Input deduplication.** `inputsEquivalent(a, b)` lets a tool tell the framework "two calls with these inputs would return the same result." The framework can then cache results across the conversation, reusing a previous answer instead of re-running.

```python
def inputs_equivalent(self, a, b):
    # File read: same path + same line range → same result (if mtime unchanged)
    return a["path"] == b["path"] and a["lines"] == b["lines"]
```

Useful for read-only, side-effect-free tools (file reads, glob matches, search). Write tools should always return `False`.

---

## Pattern 9: Versioning, Aliases, and Feature Gating

**Aliases.** A tool's name appears in the model's training data and in saved transcripts. When you rename the tool, those references break. The tool spec includes an `aliases` array:

```python
class WebFetch:
    name = "web_fetch"
    aliases = ["fetch_url", "WebFetchTool"]  # legacy names
```

The tool registry resolves any alias to the canonical tool. Old transcripts still play back; old conditioning still works.

**Feature gating.** `isEnabled()` lets a tool exclude itself from the registry based on configuration, feature flags, or experiment groups. This is the right place to ship experimental tools:

```python
def is_enabled(self):
    return self.config.get("experimental_v2_planner", False)
```

When a feature flag turns the tool off, it disappears completely — no schema in the request, no UI affordance, no permission rules.

**Strict input schemas.** Always set `strict: true` on the schema. Loose schemas silently accept extra fields, which the model then learns to send. Strict schemas reject extras at parse time, which forces the model to use the documented surface.

---

## Pattern 10: Rendering Control

Three rendering hooks decouple how the tool appears in UI from what the model sees:

| Hook | Renders | Returns `null` means |
|---|---|---|
| `renderToolUseMessage(input)` | The "tool was invoked" line | Don't show the invocation at all |
| `renderToolResultMessage(result)` | The result block in the transcript | Don't show the result at all |
| `userFacingName` | The collapsed name in the spinner | (Empty string) Don't show the tool |

These three together let you build tools that are **mechanically present but visually invisible** — they show up in the model's tool list and execute when called, but the user never sees them. This is correct for cognitive scaffolds, telemetry tools, and intra-agent coordination tools.

**Render hooks receive both input and result.** A file-edit tool can render a syntax-highlighted diff. A web-fetch tool can render the URL with a status badge. A search tool can render the matched lines with surrounding context. The renderer is a function of (input, result), not just a string.

---

## Tool Design Checklist

Use this when designing or reviewing tools:

**Capability declaration**
- [ ] Does the tool have a name that doesn't collide with other tools in the registry?
- [ ] Are there aliases registered for any previous name?
- [ ] Is `isEnabled()` wired to the right feature flag or environment check?
- [ ] Is the input schema strict (rejects extras)?

**Description hygiene**
- [ ] Does the model-facing prompt include WHEN-NOT-TO-USE with named alternatives?
- [ ] Are at least 2–3 examples included with reasoning blocks?
- [ ] Is the UI description an imperative phrase, not a name?
- [ ] Is `userFacingName` set to `""` if this tool should not appear in the UI?

**Safety profile**
- [ ] Are `isReadOnly`, `isDestructive`, `isConcurrencySafe` all functions of input, not constants?
- [ ] Does `checkPermissions` return `passthrough` for cases outside the tool's domain?
- [ ] Does `checkPermissions` canonicalize/normalize inputs via `updatedInput`?
- [ ] Is there a `toAutoClassifierInput` that produces a lossy summary for the classifier?

**Loading & discovery**
- [ ] Is the tool `shouldDefer: true` unless it's used in 50%+ of turns?
- [ ] Is `searchHint` 3–10 words, free of name-overlap, in plain capability language?

**Runtime behavior**
- [ ] Is `interruptBehavior` explicitly chosen (`cancel` for IO-bound, `block` for atomic state changes)?
- [ ] If `cancel`, does the call loop check `abortController.signal`?
- [ ] If side-effect-free, is `inputsEquivalent` implemented for caching?

**Result handling**
- [ ] Is `maxResultSizeChars` set?
- [ ] Is the truncation strategy appropriate (tail / head / middle / paginate / offload)?
- [ ] If using `newMessages`, are the injected messages justified (they will be billed on every future turn)?
- [ ] If using `contextModifier`, is the tool also marked non-concurrency-safe?

---

## Anti-Patterns Summary

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Static `isDestructive = true` on a multi-purpose tool | Forces confirmation on every safe variant; users learn to ignore | Per-input safety functions |
| Identical prompt for model and UI | Model gets too little; UI gets too much | Three text channels: prompt, description, userFacingName |
| Tool prompt with no WHEN-NOT-TO-USE | Over-selection; tool used where another is better | Explicit alternatives section with named tools |
| `checkPermissions` returns allow/deny for everything | Overreach; can't be overridden by user policy | Use `passthrough` when outside the tool's domain |
| All tools always-loaded | 15–25K wasted tokens per request | Defer most tools; load on demand via `tool_search` |
| `searchHint` repeats words from tool name | No discoverability benefit; wastes tokens | Use the *concept* the model thinks in, not the name |
| Tool returns full output regardless of size | One call can blow the context window | `maxResultSizeChars` + truncation strategy + disk offload |
| Result data and model-facing string are the same object | Either rich data is sent to model (token waste) or model gets a string that analytics can't parse | Separate `data` payload from `mapToolResultToToolResultBlockParam` projection |
| Injected `newMessages` not justified | History grows unboundedly | Inject only when the message is needed in *every* future turn |
| `contextModifier` on a concurrency-safe tool | Two parallel calls produce ambiguous merged context | Mark the tool non-concurrency-safe when it modifies context |
| Loose schema with extra fields | Model learns to send undocumented parameters; cleanup is painful | `strict: true` always |
| Renamed tool without alias | Old conditioning breaks, transcripts replay incorrectly | Always add an alias on rename |
| No `interruptBehavior` declared | User interrupt creates dangling state or stale partial output | Pick `cancel` or `block` explicitly per tool |
| Cognitive instruction in prose, repeated in system prompt | Model ignores it under pressure | Promote the instruction to a cognitive-scaffold tool |
