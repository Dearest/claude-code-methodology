# System Prompt Architecture Patterns

---

## The Core Insight

The system prompt is **not a string**. It is an **ordered list of named, individually-cacheable sections** assembled at the start of each turn, with a strict boundary between content that is identical across all sessions (cacheable) and content that is specific to this session (not cacheable).

Every architectural decision in this dimension is downstream of one fact: the API charges full price for the volatile suffix and a fraction of the price for the stable prefix. At one user, this is invisible. At ten thousand users, the difference between a well-architected prompt and a naive one is the difference between affordable and not.

But cost is only one of three goals. The same architecture also delivers:

- **Maintainability** — each section is independently authored, tested, and replaceable
- **Per-audience adaptation** — internal users, external users, autonomous agents, and headless modes share most sections but vary a few
- **Cache stability across feature flips** — feature flags that affect output format are deliberately structured so a flip does not invalidate the entire prefix hash

---

## Pattern 1: The Boundary Marker

A literal sentinel string — typically `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` — separates the cacheable prefix from the volatile suffix. The API client recognizes this marker and asks the model service to cache everything up to (but not including) the line where it appears.

```
[Static section 1: introduction]
[Static section 2: system rules]
[Static section 3: task guidelines]
[Static section 4: action safety]
[Static section 5: tool usage]
[Static section 6: tone and style]
[Static section 7: output efficiency]
__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__
[Dynamic section 1: session guidance]
[Dynamic section 2: memory]
[Dynamic section 3: environment info]
[Dynamic section 4: language]
[Dynamic section 5: output style]
[Dynamic section 6: MCP instructions]
[…]
```

**The boundary is a hard contract.** Anything before it must be invariant across all users, all sessions, all sub-agents of the same type. Anything that *could* vary by session — current date, working directory, enabled tools, feature flags that change copy — belongs after the boundary.

**Two common mistakes that bust the cache:**

1. **Putting "today's date" in the intro.** The intro looks static, but the date changes daily, so every request after midnight invalidates the cache. Move it to the env-info dynamic section.

2. **Putting a feature-flag check in a static section.** Even if the section text is identical for both flag values, the gating logic might select between two different *whole sections*, doubling the number of prefix-hash variants in your cache fleet. Either move the gated content past the boundary or design the gate so both branches produce identical bytes.

---

## Pattern 2: Section as a First-Class Object

Each section is an object, not a string:

```python
class Section:
    name: str
    compute: Callable[[], str | None]
    cache_break: bool   # if True, recomputed every turn
```

There are two factory functions, distinguished by name:

```python
def section(name, compute) -> Section:
    return Section(name, compute, cache_break=False)

def DANGEROUS_uncached_section(name, compute, reason: str) -> Section:
    # reason is mandatory — forces the author to justify cache-breaking
    return Section(name, compute, cache_break=True)
```

**The `DANGEROUS_` prefix is a design choice, not a flourish.** It makes the cost visible at the call site. Reviewers can grep for it to audit every place that breaks the cache. The mandatory `reason` argument forces the author to articulate why this section can't be memoized.

**Memoization scope.** Memoized sections are computed once per session and cached until `/clear` or `/compact`. Within a session, repeated turns reuse the cached value. This matters because some sections are expensive to compute — loading memory files from disk, scanning the project tree, querying a remote settings service.

**Parallel resolution.** When the agent assembles a prompt, all section compute functions run concurrently (via `Promise.all` or equivalent). Sections must be independent — a section cannot depend on another section's output. If you need cross-section coordination, do the coordination in a shared upstream computation that each section reads.

---

## Pattern 3: Static and Dynamic Section Inventories

The exact section names will vary per agent, but the canonical inventory looks like this:

**Static (before boundary):**

| Section | Content |
|---|---|
| `intro` | Role definition, security declaration, mission statement |
| `system_rules` | Operating rules (tool use, formatting, permissions, file paths) |
| `task_guidelines` | How to do tasks (over-engineering warnings, no premature abstraction) |
| `actions` | Reversibility and confirmation patterns for risky actions |
| `tool_usage` | Prefer dedicated tools over general ones; selection guidance |
| `tone_style` | Communication norms (emoji, code references, markdown) |
| `output_efficiency` | Token-conscious output rules |

**Dynamic (after boundary):**

| Section | Recompute trigger | Why dynamic |
|---|---|---|
| `session_guidance` | Per-session tool enablement | Different tools → different guidance |
| `memory` | Loaded from disk at session start | Memory files change between sessions |
| `env_info` | Per-session env capture | CWD, git branch, OS, date all vary |
| `language` | User language setting | Personalization |
| `output_style` | Output style config | Persona swap |
| `mcp_instructions` | **Uncached** (servers connect/disconnect mid-session) | MCP servers can come and go between turns |
| `scratchpad` | Scratchpad directory info | Path varies |
| `frc` (function result clearing) | Per-model rules for tool result handling | Model-dependent |

**`mcp_instructions` is the canonical example of a justified `DANGEROUS_uncached` section.** The reason: an MCP server can connect or disconnect between turns; the prompt must reflect the current set. If memoized at session start, late-connecting servers would have no instructions in the prompt for the rest of the session.

---

## Pattern 4: Modular Sections Beat Monolithic Strings

Each section function has a single responsibility and returns a string (or `null` to omit). This is not just tidiness — it enables:

1. **Independent unit tests.** Each section is testable without rendering the full prompt.
2. **Per-audience variants.** Inject a different content for internal users by swapping a single section function, not by maintaining two versions of the whole prompt.
3. **Conditional inclusion.** A section returns `null` when not applicable (no MCP servers connected → no MCP instructions section). The filter removes nulls before joining.
4. **Per-section caching.** Each section has its own cache entry; expensive sections don't force re-computation of cheap ones.
5. **Section replacement for agent variants.** A research agent can swap `task_guidelines` for `research_guidelines` by passing a different section function — every other section stays identical and stays cached.

**The "null section" pattern.** Returning `null` from a section function is how you express "this section doesn't apply this session." The framework filters nulls before joining:

```python
sections = [s for s in resolved_sections if s is not None]
return "\n".join(sections)
```

This is much cleaner than wrapping conditionals around each section call site, and it preserves the section order so a section that turns on mid-session lands in the right slot.

---

## Pattern 5: Conditional Bullets Inside a Section

Below the section level, the same `null`-filtering pattern applies to bullet lists. A guidance section might look like:

```python
def session_guidance(enabled_tools, skill_commands):
    items = [
        "If you do not understand why the user denied a tool call, "
        "use the AskUserQuestion tool to ask."
        if "AskUserQuestion" in enabled_tools else None,

        "If you need the user to run a shell command themselves, "
        "suggest they type `! <command>` in the prompt."
        if not is_non_interactive_session() else None,

        get_agent_tool_section() if "Agent" in enabled_tools else None,

        f"/<skill-name> invokes a user-invocable skill via the Skill tool."
        if skill_commands else None,
    ]
    items = [i for i in items if i is not None]
    if not items:
        return None  # whole section omits
    return "# Session-specific guidance\n" + format_bullets(items)
```

**Why this matters.** The session-guidance section's bullets depend on which tools are enabled this session. If you put this content in the static prefix, every distinct combination of enabled tools produces a different prefix hash, multiplying cache variants by 2^N where N is the number of optional tools. Keep this kind of branching content in the dynamic suffix; the comment in real production code about this is "would otherwise multiply the prefix hash variants (2^N)".

---

## Pattern 6: Audience Variants

A production agent serves more than one audience. The same agent might be used by:

- Public external users (most conservative defaults)
- Internal employees (more detailed communication norms, additional debugging instructions)
- Autonomous background agents (no user to ask, must be fully self-directed)
- Headless API consumers (no UI, no permission dialogs)

The mechanism is a small number of environment variables or settings flags that section functions consult:

```python
def tone_and_style():
    base = [
        "Only use emojis if the user explicitly requests it.",
        "When referencing functions, use file_path:line_number.",
    ]
    if os.environ.get("USER_TYPE") == "internal":
        # Internal users get richer guidance on prose, less on terseness
        base.extend([
            "Write user-facing text in flowing prose, not fragments.",
            "Expand technical terms; err on the side of more explanation.",
        ])
    else:
        # External users get the terseness reminder
        base.append("Your responses should be short and concise.")
    return "# Tone and style\n" + format_bullets(base)
```

**Audience flags belong inside sections, not around them.** Two parallel section sets ("internal prompt" vs "external prompt") rapidly diverge and drift. Variant logic inside each section keeps the structural skeleton shared.

**Audience flags are not features for users.** They are operator-controlled. A user cannot escalate themselves to the internal variant; only the operator (or the binary build) can.

---

## Pattern 7: Section Sourcing — Where the Content Comes From

A section's compute function pulls its content from one of four sources:

| Source | Examples | Caching characteristics |
|---|---|---|
| **Constant string** | Tone-and-style, output-efficiency | Compute is trivially cheap; cache is for cleanliness, not perf |
| **Disk read** | Memory files, project config | Expensive; cache aggressively, invalidate only on `/clear` |
| **System probe** | env_info (CWD, git branch, date) | Cheap; recompute per session is fine |
| **Settings/feature lookup** | language, model overrides | Cheap; cache between turns |

**Don't make the model wait on remote calls.** Section compute is on the critical path of every turn. A section that fetches from a remote service blocks the API request. If you need remote data, pre-fetch into a local cache out-of-band and have the section read the cache.

---

## Pattern 8: customSystemPrompt vs appendSystemPrompt

Two configuration knobs let consumers customize the prompt without forking your section list:

- **`customSystemPrompt`** — *replaces* the entire system prompt with a user-supplied string. The user gets full control and pays full token cost. Used by SDK consumers who want a different agent persona entirely.
- **`appendSystemPrompt`** — *appends* a user-supplied string after the framework's prompt. The framework's prompt still runs (including the boundary, including the dynamic sections). The appended content lands in the volatile suffix.

The semantic difference matters for support:

- A bug report from a `customSystemPrompt` user is opaque — the framework knows nothing about what's in their prompt.
- A bug report from an `appendSystemPrompt` user is debuggable — the framework's prompt is intact and the append is small and visible.

**Guidance for users:** prefer `appendSystemPrompt` for tweaks; reserve `customSystemPrompt` for genuinely different agents.

**Caching consequence:** `customSystemPrompt` breaks the framework's cache for that user; `appendSystemPrompt` does not (the framework's prefix is unchanged; the user's appended bytes just push the boundary later).

---

## Pattern 9: Output Style as a Section Variant

An "output style" is a named persona that swaps the intro section and the tone-style section, while keeping everything else. The user picks a style; the prompt adapts.

```python
output_styles = {
    "default":   {"intro": default_intro, "tone": default_tone},
    "explanatory": {"intro": teacher_intro, "tone": verbose_tone},
    "minimalist": {"intro": terse_intro, "tone": one_line_tone},
}
```

The crucial design constraint: each output style is **a fixed set of section bytes**, not a parameterized generator. Two users who pick the same style get the same prefix bytes, so they share cache. If styles were parameterized ("verbose level 7"), each parameter value would be a separate cache key.

**Where styles live.** Output styles can be defined in code, loaded from disk (so users can author their own), or supplied via SDK. The framework reads the style at session start and either:

- Treats the style as part of the static prefix (cache per style) — if the style choice doesn't change mid-session
- Treats the style as a dynamic section (cache by style name, fall through to default) — if the user can change style mid-session

Either choice is defensible; pick one and document the constraint.

---

## Pattern 10: Memoization Through `/clear` and `/compact`

The framework provides two user commands that wipe the prompt section cache:

- **`/clear`** — the user wants a fresh conversation. Drop history, drop cached sections, recompute everything.
- **`/compact`** — the framework just compressed the conversation. The dynamic sections (especially memory and env_info) may now be stale or worth re-extracting. Drop cached sections.

When the cache is cleared, the next turn pays the full computation cost (loading memory, scanning project, etc.) — but it also paths back into a "fresh prefix" that the API caches anew. The cache invalidation cost is one slow turn, recouped over the next many cached turns.

**Provide an explicit "clear cache" hook.** Beyond `/clear` and `/compact`, internal subsystems (memory extraction, settings change, feature flag flip) may want to invalidate specific sections. Expose a per-section cache invalidation API rather than forcing them to invalidate the whole cache.

---

## System Prompt Design Checklist

**Architecture**
- [ ] Is there a single boundary marker, clearly named, and not present in any compute output?
- [ ] Can you list which sections are before the boundary and which are after, without reading the code?
- [ ] Does every section function have a single responsibility?
- [ ] Do sections compute in parallel, with no cross-section dependencies?

**Caching**
- [ ] Are all uncached sections marked with the `DANGEROUS_` prefix?
- [ ] Does each uncached section have a documented reason?
- [ ] Are feature-flag gates either past the boundary, or producing identical bytes across flag values?
- [ ] Are date/time, CWD, git branch, and anything else session-specific *only* in dynamic sections?

**Variants**
- [ ] Are audience variants (internal/external/autonomous) expressed via conditional content inside sections, not parallel prompt sets?
- [ ] Are output styles a fixed set of named byte combinations, not parameterized generators?
- [ ] Is `appendSystemPrompt` supported for user tweaks?
- [ ] Is `customSystemPrompt` documented as breaking the framework's cache?

**Cleanup**
- [ ] Are there explicit cache-invalidation hooks for `/clear`, `/compact`, settings change?
- [ ] Is memoization scoped per-session, not per-process (so multi-tenant servers don't cross-pollute)?

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Monolithic single-string prompt | Untestable, uncacheable, unmaintainable | Modular sections |
| Date/time in the intro section | Daily cache invalidation across all sessions | Move to env-info dynamic section |
| Feature flag check that selects between whole static sections | 2^N prefix-hash variants in the cache fleet | Either past-boundary, or identical-byte branches |
| `DANGEROUS_uncached` without a documented reason | Silent perf regression; reviewers can't audit | Mandatory `reason` argument |
| Cross-section dependency (section B reads section A's output) | Forces sequential compute; one slow section blocks the prompt | Compute shared values upstream, pass to both sections |
| Per-audience parallel prompt sets | Drift between variants; bug fixes land in one and not the other | Variant logic inside shared section functions |
| Output style as a free-form template | Each style instance is a different cache key | Fixed named styles with fixed section bytes |
| Section reads from remote service inline | Blocks API request on remote latency | Pre-fetch into local cache out-of-band |
| `customSystemPrompt` without warning about cache | Users surprised by cost; framework's prompt vanishes silently | Document that custom replaces, append augments |
| No `/clear` semantics | Stale cached sections persist after intended reset | Explicit cache-invalidation API tied to `/clear` and `/compact` |
| Session-guidance content baked into static prefix | Tool-set variations explode the cache space | Session guidance is dynamic; static section enumerates capability families, not specific tools |
