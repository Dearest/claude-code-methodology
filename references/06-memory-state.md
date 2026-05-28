# Memory and State Patterns

---

## The Core Insight

A production agent must distinguish between **state with different lifetimes and access patterns**. Conflating them — treating session state as if it were memory, or memory as if it were configuration — produces either data loss (the state was too volatile) or stale data (the state was too persistent).

Five distinct categories exist in a mature system:

| Category | Lifetime | Where it lives | Access pattern |
|---|---|---|---|
| **Turn state** | Single API call | In-message | Read once, write once, then frozen |
| **Session state** | Conversation duration | In-memory, possibly persisted | Mutated frequently |
| **Persistent memory** | Across sessions | Disk, file-based | Loaded at session start, written occasionally |
| **Team memory** | Across users/sessions in a team | Disk, synced | Read at session start, writes propagated |
| **Configuration** | Across many sessions, all of a kind | Disk, layered files | Read at startup, rarely changes |

The patterns below treat each category as a separate problem with its own primitives. Mixing them in implementation creates bugs that are extremely hard to diagnose because they violate invariants the developer never wrote down.

---

## Pattern 1: AppState as the Session Store

The session-scoped state is a single object — `AppState` — with all the things that change during a conversation:

```typescript
type AppState = {
  messages: Message[]
  todos: Record<TodoKey, Todo[]>
  toolDecisions: Map<ToolUseId, ToolDecision>
  fileStateCache: Map<Path, FileState>
  inProgressToolUseIds: Set<string>
  permissionContext: ToolPermissionContext
  costTracking: CostTracking
  // ...
}
```

**Updates are immutable.** Every write produces a new `AppState`; the old one is unchanged. The pattern is React-style:

```python
def set_app_state(updater):
    new_state = updater(get_app_state())
    store.set(new_state)
```

```python
set_app_state(lambda prev: prev.with_message(new_message))
```

**Three reasons immutable updates matter:**

1. **Debugging.** A bug becomes "AppState at turn N had `X`, AppState at turn N+1 had `Y`, why?" — you can replay the sequence of updates.
2. **No async races.** When two async operations both call `set_app_state`, the updater function lets the framework apply them in a defined order without losing one.
3. **Subagent isolation.** Subagents can get a snapshot of the parent's AppState, run independently, and merge specific changes back without their internal mutations leaking.

**Async subagents need a separate hook.** For sub-agents whose `setAppState` is a no-op against the parent store (because they run on a separate event loop), the framework exposes `setAppStateForTasks` — a *root-reaching* setter for infrastructure that outlives a single turn (background tasks, session-level hooks). The subagent can register itself in the parent's task registry even though its other state changes don't propagate. This distinction is subtle but important: ordinary writes are scoped; infrastructure writes are global.

---

## Pattern 2: File State Cache

Files are read many times across a session. Without caching, every read costs IO and the framework cannot detect external modifications. With a cache:

```python
class FileStateCache:
    entries: Dict[Path, FileState]  # LRU-bounded
    MAX_ENTRIES = 1000

    @dataclass
    class FileState:
        content_hash: str
        last_read_at: datetime
        last_modified_by_us: bool
        mtime_on_disk: float

    def read(self, path):
        entry = self.entries.get(path)
        if entry and not self._mtime_changed(path, entry):
            return entry.content
        # File changed externally, or never read; do real read
        content = self._read_from_disk(path)
        self._store(path, content)
        return content

    def write(self, path, content):
        self._write_to_disk(path, content)
        self._store(path, content, last_modified_by_us=True)
```

**Three behaviors the cache enables:**

1. **External modification detection.** Before editing a file, compare the cache's `content_hash` to the file's current state. If they differ, the file was modified externally (another editor, a build tool, another agent) — refuse the edit, prompt the user.

2. **Read deduplication.** Re-reading the same file in the same session returns cached content. The framework still verifies mtime; only the IO is saved.

3. **Audit trail.** A list of files the agent has read or written, with timestamps, is the foundation of "what did the agent do this session?"

**Bound the cache.** A long session may touch thousands of files; an unbounded cache slowly leaks memory. LRU eviction with a generous cap (a few thousand entries) is fine — evicted entries get re-read at small cost.

**Caveat: the cache is also a deduplication keyset for attachments.** When CLAUDE.md or other nested-memory files are injected as attachments, the framework checks the cache to avoid injecting the same file twice. Because the cache is LRU, busy sessions can evict the entry and re-inject. Maintain a separate `loadedNestedMemoryPaths` set for these injections, distinct from the general read cache.

---

## Pattern 3: Tool Decision Tracking

Every permission decision should be persisted, indexed by the tool_use_id of the call it gated:

```python
toolDecisions: Map[ToolUseId, ToolDecision]

@dataclass
class ToolDecision:
    source: str           # 'classifier' | 'user' | 'rule' | 'hook'
    decision: str         # 'accept' | 'reject'
    reason: DecisionReason  # structured, see permission-safety doc
    timestamp: datetime
    input_summary: str    # lossy summary for analytics
```

This decision record supports:

- **Audit:** "what permissions did the agent ask for this session?" is one query
- **Debugging:** when a tool was unexpectedly denied, you can trace exactly which decision and why
- **Replay:** given the decisions, you can re-derive what the agent did
- **Suggestion engine:** the framework can notice "user denied this kind of call 5 times" and proactively suggest a rule

---

## Pattern 4: Persistent Memory — The Memdir Pattern

For information that should outlive a session, the established pattern is **memdir**: a directory of typed markdown files, with one index file, organized semantically.

**Directory layout:**

```
memory/
├── MEMORY.md                  ← the index
├── user_role.md
├── user_preferences.md
├── feedback_testing.md
├── feedback_style.md
├── project_q4_initiative.md
├── reference_dashboards.md
└── ...
```

Each memory file is a standalone markdown document with frontmatter:

```markdown
---
name: feedback-testing
description: Integration tests must hit a real database, not mocks
metadata:
  type: feedback
---

Integration tests must hit a real database, not mocks.

**Why:** Prior incident — mock/prod divergence masked a broken migration.

**How to apply:** Whenever writing tests that touch DB code, run against a real DB instance; mock only stable external services (third-party APIs).
```

**MEMORY.md is the index, not a memory.** It is a one-line-per-entry pointer file:

```markdown
- [User role](user_role.md) — data scientist, observability-focused
- [User preferences](user_preferences.md) — terse responses, no trailing summaries
- [Testing feedback](feedback_testing.md) — integration tests need real DB
- ...
```

The index is **always loaded into context** at session start; individual memory files are read on demand when referenced. This achieves progressive disclosure: the model sees what memories exist without paying full cost for their content.

**Bounded entrypoint.** `MEMORY.md` is enforced to fit within both a line cap (`MAX_ENTRYPOINT_LINES = 200`) and a byte cap (`MAX_ENTRYPOINT_BYTES = 25_000`). Excess is truncated with a warning. Without this, MEMORY.md grows unbounded and silently consumes context.

---

## Pattern 5: The Four Memory Types

Memdir constrains memories to four types, each with a different purpose, contract, and prompt template:

| Type | Purpose | When to save | Frontmatter `metadata.type` |
|---|---|---|---|
| **user** | Who the user is — role, expertise, preferences | When learning about user's role, preferences, knowledge | `user` |
| **feedback** | How to approach work — what to do, what to avoid | When user corrects you OR confirms an unusual choice worked | `feedback` |
| **project** | Ongoing work context — initiatives, deadlines, incidents | When learning who-is-doing-what-and-why; not derivable from code | `project` |
| **reference** | Pointers to external systems | When learning about external sources of truth | `reference` |

**These four are not arbitrary.** The taxonomy was reduced from more types over time because most additional categories ended up being collapsible into one of these. Adding a fifth invites drift — memories get classified inconsistently and the model can't tell what's where.

**Anti-pattern memories (do NOT save):**

- Code patterns, conventions, architecture, file paths — derivable from `git`, `grep`, project files
- Git history, recent changes, blame — `git log` is authoritative
- Debugging recipes — the fix is in the code; commit messages have the context
- Anything already documented in CLAUDE.md or equivalent
- Ephemeral task details: in-progress work, current conversation context

The rule of thumb: **save what is non-obvious to a fresh agent reading the project, and what would otherwise have to be re-discovered through conversation.**

---

## Pattern 6: Memory Body Structure

For **feedback** and **project** memories, the body follows a fixed structure:

```
{Lead with the rule or fact in one sentence.}

**Why:** {The motivation — often a past incident, a deadline, a stakeholder ask, a strong preference.}

**How to apply:** {When and where this guidance kicks in. Specific enough to act on.}
```

**Why this structure matters.** The `Why` line is the most important. Without it, the agent applies rules mechanically and gets edge cases wrong. With it, the agent can reason: "this rule says X because of Y; in this edge case Y doesn't apply, so I should relax X."

For **user** and **reference** memories, the body is freer-form — they describe a state of the world, not a directive.

---

## Pattern 7: Inter-Memory Links

Memories can reference each other:

```markdown
This guidance is related to [[feedback-testing]] — the same prior incident motivated both rules.
```

The `[[name]]` syntax targets the `name:` field in another memory's frontmatter. The framework does not enforce that the target exists — a dangling link is just a marker for future authoring ("write a memory about this when you have one").

**Why links matter.** Memories cluster. A user's preference for terse responses (`user_preferences`) might be related to their preference for bundled PRs (`feedback_pr_strategy`). The links surface these relationships to the model so it can apply them together.

**Avoid hierarchies via filenames.** Don't put memories in subdirectories to express categorization. The semantic relationship is the link, not the path. Subdirectories make filenames ambiguous and break linkability.

---

## Pattern 8: Memory Extraction is Triggered, Not Manual

Users will not curate their own memory directory. Anything that requires manual upkeep degrades to nothing within a week.

The pattern: **the framework decides when to save, what to save, and how to format it.** The user doesn't issue a save command. Instead, the framework watches for triggering signals:

- User correction ("no, not like that") → feedback memory
- User confirmation of an unusual choice ("yes, exactly that") → feedback memory
- User describes their role or expertise → user memory
- User mentions a deadline, project context, recent decision → project memory
- User references an external system as authoritative → reference memory

When a signal fires, the framework saves the memory (or updates an existing one). The user does not have to think about it.

**Confirmations are quieter than corrections — watch for them.** A user saying "yeah, that's right" after the agent chose an unusual approach is *more* informative than a correction, because it confirms a non-obvious judgment call. Many systems only learn from corrections and progressively drift toward over-caution. A good extractor learns from both.

**Always cite the trigger in the saved memory.** "Confirmed after I chose this approach — a validated judgment call, not a correction" — this metadata helps future-you (or future-agent) judge whether the memory is still load-bearing if circumstances change.

---

## Pattern 9: Staleness and Verification Before Use

Persistent memory is frozen in time. A memory saved when the project was in Phase 1 may be wrong by Phase 3. A memory that names a function or file is a claim that the artifact existed *when the memory was written*.

**Rules for using a recalled memory:**

1. **A memory naming a file path:** verify the file still exists before relying on it.
2. **A memory naming a function or flag:** grep for it before recommending it.
3. **A memory naming a date or deadline:** check whether the date has passed.
4. **A memory describing recent activity:** consider it potentially stale; prefer `git log` for *current* state.

When a recalled memory conflicts with the current state of the world, **trust the current observation, not the memory.** Update or delete the stale memory rather than acting on it.

**The framework can prompt for stale-memory cleanup.** At session start, if any memory references a file that no longer exists or a date long-passed, mark it for review. Don't auto-delete (the user might want to keep it as historical context), but surface it.

---

## Pattern 10: Private vs Team Memory

Some memory is for the individual user; some is for the project/team. The same agent often needs both:

```
~/.agent/memory/                 ← private (this user, all projects)
{project-root}/.agent/memory/    ← team (this project, all users)
```

The team directory is committed to the project repo and synced via normal version control. Memories there are visible to every contributor.

**Scope guidance per memory type:**

| Type | Default scope |
|---|---|
| **user** | Always private |
| **feedback** | Default private; team only if the guidance is a project-wide convention every contributor should follow |
| **project** | Strongly bias toward team |
| **reference** | Usually team (external systems are typically project-wide) |

**Conflict resolution.** When a private memory contradicts a team memory, the team memory wins by default. Either don't save the private one, or note the override explicitly: "This is my personal style override of the team's `feedback-testing` policy."

**Team memory and code review.** Treat team memory writes like code commits — they should be reviewable. A PR that adds a new team memory is welcome; one that adds a memory the team disagrees with should be rejected.

---

## Pattern 11: Append-Only Logs vs Curated Memory

For some kinds of information, a curated memory directory is wrong — the right pattern is an append-only log:

- Daily standups
- Decision logs ("today we chose X over Y because ...")
- Incident postmortems
- Activity summaries

For these, maintain a date-named log file (`2026-01-15.md`, `2026-01-16.md`, ...) and append rather than mutate. MEMORY.md doesn't list every log file; it lists the *log directory* and the model reads recent ones as needed.

**The boundary:** if a piece of information is "this is true now" → curated memory file. If it is "this happened on date X" → append-only log entry. Don't put events in memory files; don't put state in logs.

---

## Pattern 12: Hierarchical Configuration vs Memory

Config and memory look similar (both are persistent, both shape behavior). They are different:

| Property | Configuration | Memory |
|---|---|---|
| Authored by | Operator / developer / power user | Framework, on signal |
| Lifetime | Long-term, stable | Variable, decays |
| Granularity | Coarse (per-project, per-user) | Fine (per-fact) |
| Mutability | Edits are deliberate, version-controlled | Auto-extracted, auto-updated |
| Example | "Default model is Opus" | "User is a data scientist focused on observability" |

A production agent has both, layered:

```
/etc/agent/policy.yaml                  ← operator-managed, bypass-immune
~/.agent/config.json                    ← global user config
{project-root}/.agent/config.json       ← per-project config
{project-root}/.agent/memory/           ← team memory directory
~/.agent/memory/                        ← private memory directory
{project-root}/CLAUDE.md                ← project instructions (in-tree)
{subdirectory}/CLAUDE.md                ← scope-narrower instructions
```

**Each layer can add, never remove.** Config and memory both accumulate by layer; lower layers add specificity, they don't override safety properties of higher layers.

---

## Memory and State Design Checklist

**Session state**
- [ ] Is there a single `AppState` object holding all session-scoped state?
- [ ] Are updates immutable (updater functions, not mutations)?
- [ ] Is there a separate hook for infrastructure that outlives a turn (background tasks)?

**File state**
- [ ] Is there a file state cache with content hashes and mtimes?
- [ ] Does the framework detect external modifications before write?
- [ ] Is the cache LRU-bounded to prevent leaks?
- [ ] Is there a separate dedup set for nested-memory attachments?

**Tool decisions**
- [ ] Is every permission decision persisted with a structured reason?
- [ ] Is the decision indexed by tool_use_id for traceability?

**Persistent memory**
- [ ] Is memory file-based (not database-locked, not config-locked)?
- [ ] Does each memory file have a frontmatter type from a closed taxonomy?
- [ ] Is there an index file (`MEMORY.md`) loaded at session start?
- [ ] Is the index bounded by line count and byte count?
- [ ] Are memory contents loaded on demand, not all upfront?

**Memory authoring**
- [ ] Are memories framework-extracted from triggering signals, not manually saved?
- [ ] Do feedback memories include the `Why:` line?
- [ ] Are confirmations (not just corrections) recognized as save triggers?
- [ ] Are relative dates converted to absolute on save?

**Memory hygiene**
- [ ] Does the framework verify file/function/path references in memories before relying on them?
- [ ] Are stale-looking memories surfaced for review at session start?
- [ ] Are there explicit memory-removal triggers (user says "forget X")?

**Scope and sharing**
- [ ] Is there a clean separation between private and team memory directories?
- [ ] Does the team memory directory live in the project repo (version-controlled)?
- [ ] Are conflicts between private and team memory resolved with the team winning by default?

**Configuration**
- [ ] Is configuration layered (operator → user-global → project → subdirectory)?
- [ ] Is the hierarchy clear and documented?
- [ ] Can config layers add but not weaken safety properties of higher layers?

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Global mutable state | Async races, hard to debug | Immutable AppState with updater functions |
| File reads with no cache | Repeated IO; misses external modifications | File state cache with mtime and hash |
| All memories loaded into context at session start | Bloat; sessions start expensive | Index + progressive disclosure |
| `MEMORY.md` grows unbounded | Silent context consumption | Enforce line and byte caps |
| Memory of code patterns and architecture | Duplicates project state; goes stale fast | Don't save what `grep` can find |
| Memory without a `Why:` | Mechanical rule-following; bad edge-case judgment | Mandatory `Why:` for feedback and project memories |
| Manual save commands | Users forget; system stays empty | Framework extracts on signal |
| Only learning from corrections | Drift toward over-caution | Watch for confirmations too |
| Relative dates in memories | Memory becomes meaningless six months later | Always convert to absolute on save |
| Memories never verified | Confidently recommend non-existent files and removed functions | Verify references before recommending |
| Memory taxonomy growing over time | Inconsistent classification; model can't tell what's where | Keep the type set small and stable |
| Mixing event logs and memory files | Event logs grow forever; memory should be curated | Separate append-only logs from curated memories |
| Config and memory in the same file | Updates conflict, layering breaks | Separate config files from memory directory |
| Subdirectories for memory categorization | Breaks `[[link]]` resolution; ambiguous names | Flat directory, link via frontmatter `name` |
| Private memory shadowing team memory silently | Confusing behavior, hard to debug | Explicit override note, or refuse the private save |
