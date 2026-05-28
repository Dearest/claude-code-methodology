# Context and Token Economy Patterns

---

## The Core Insight

Token usage is a first-class engineering concern, in the same way that memory is for systems programming. You think about allocation, lifetime, sharing, eviction, and proactive reclaim. Bad token economics do not matter at prototype scale; at production scale they determine whether the product is viable.

A production agent has **four token pools**, each with different management requirements:

| Pool | What it contains | Lifetime | Management strategy |
|---|---|---|---|
| **System prompt prefix** | Identity, rules, tone | Process lifetime | Cache aggressively, keep byte-stable |
| **System prompt suffix** | Memory, env, session config | Session lifetime | Recompute per session, cache within session |
| **Tool schemas** | Tool definitions | Per request | Defer everything that isn't core |
| **Conversation body** | Messages, tool results, attachments | Conversation lifetime | Truncate, compress, evict, summarize |

The patterns below treat each pool separately. Conflating them — applying a single "reduce tokens" heuristic across all four — leaves significant savings on the table and introduces regressions in the wrong places.

---

## Pattern 1: The Effective Context Window Is Smaller Than the Model's Limit

The model's advertised context window is not the budget you actually have. Subtract:

```
effective_context = model_window
                  − output_reserve              (≈ 20K for compact summary)
                  − safety_buffer               (≈ 13K)
                  − operator_override_if_any    (CLAUDE_CODE_AUTO_COMPACT_WINDOW)
```

The output reserve guarantees the model has room to generate a compaction summary when triggered. Without it, you reach the limit, try to compact, and the compaction itself fails because there is no space for its output. This is one of the worst classes of bug because it fires only at the worst moment.

The safety buffer absorbs unpredictable variance — token-counting is approximate; one expensive tool result can blow through 5K unexpectedly.

**Operator-level overrides matter.** Power users want to simulate small windows to test their compaction logic; production deployments may want smaller effective windows to reduce per-request cost. The framework respects a `CLAUDE_CODE_AUTO_COMPACT_WINDOW` env var that *lowers* the effective ceiling (it should never raise it past the model's actual limit).

---

## Pattern 2: Threshold Trio for Window Pressure

A single "compact when full" trigger is too coarse. The framework defines three thresholds, all measured in tokens remaining before the effective ceiling:

| Threshold | Buffer | Trigger |
|---|---|---|
| **Warning** | ~20K remaining | Show user a yellow indicator ("approaching context limit") |
| **Auto-compact** | ~13K remaining | Trigger compaction at the end of the current turn |
| **Manual compact** | ~3K remaining | User-initiated `/compact` becomes available |

The progression provides graceful degradation:

- The user sees the warning first and can wrap up if they choose
- If they don't, the auto-compact fires proactively before the wall
- If even that fails, manual compact remains until the very edge

**The buffer sizes are calibrated to compaction summary size.** The output reserve (20K) is the p99.99 of summary output length. The warning threshold matches the reserve because once you cross it, a compaction is no longer guaranteed to fit. The auto-compact threshold sits a bit further in to leave room for one more turn before compaction is mandatory.

---

## Pattern 3: Four Compaction Strategies, Different Triggers

There is no single "compaction" mechanism. There are at least four, each appropriate for a different situation:

### Auto-compact (full conversation summary)

- **Trigger:** approaching the context window ceiling (auto-compact threshold)
- **Mechanism:** summarize the entire conversation history into a single dense summary; replace the old history with the summary; continue
- **When the model regains control,** it has the summary plus a few re-attached "anchor" pieces of recent content
- **Cost:** one expensive model call to produce the summary; loss of detail in the discarded history

### Micro-compact (tool-result-level compression)

- **Trigger:** individual tool results growing too large within an otherwise fine conversation
- **Mechanism:** replace old tool results with placeholders like `[Result evicted to save context — was 47K chars]`
- **Preserves:** the message structure, the tool_use blocks, the relationship between calls and results — only the *result content* is compressed
- **When to prefer:** when only a few specific results are oversized, not when the whole history is too long

### Snip-compact (head-truncate for retry)

- **Trigger:** an API call returns "prompt too long" deterministically
- **Mechanism:** truncate the head of the conversation (oldest messages) by a target token count and retry
- **When to prefer:** as a last-resort recovery when an unexpected jump in conversation size hits the wall mid-turn
- **Important:** this is *not* a planned compaction; it's emergency surgery

### Session-memory compact (extracted to disk)

- **Trigger:** session ending, or context approaching limits with extractable memories
- **Mechanism:** identify facts worth remembering (user preferences, project context, ongoing work) and persist them to disk; remove the source messages from active context
- **Preserves the knowledge across sessions** — the next session reloads from disk via the memory section
- **When to prefer:** any time a long-running project conversation needs to compact, this beats auto-compact because the extracted memories survive

A production agent has *all four*. Auto-compact is the headline mechanism. Micro-compact and snip-compact are emergency surgeries. Session-memory compact is the long-term solution for project continuity.

---

## Pattern 4: The Compact Boundary Marker

Compaction does not simply replace history. It produces a structured `compact_boundary` system message that:

1. Marks the end of the old (now-summarized) history
2. Carries metadata about what was preserved and what was compacted
3. Lets resume code reconstruct the post-compact state

```typescript
type CompactBoundaryMessage = {
  type: 'system'
  subtype: 'compact_boundary'
  compactMetadata: {
    preservedSegment?: {
      tailUuid: string             // Last preserved message ID
      // ...the messages between this and the boundary survive
    }
    summaryText: string            // The compact summary
    triggerReason: 'auto' | 'manual' | 'micro' | 'session_memory'
    tokensBeforeCompact: number    // For telemetry
    tokensAfterCompact: number
  }
}
```

**The boundary is durable.** When the session is saved and resumed, the resume code reads the boundary, drops anything before `preservedSegment.tailUuid`, and reconstructs a coherent post-compact state. Without the boundary, resuming a compacted session would replay the pre-compact history and re-compact, doubling cost.

**The boundary also gates SDK consumers.** When an SDK consumer streams a session, the boundary tells them "from here, the prior content is summarized; do not assume it is in the model's context."

---

## Pattern 5: Post-Compaction Restoration

After compaction, the model loses most of its specific context. A naive design just leaves it with the summary and resumes the conversation. The user immediately asks "what was that file we were looking at?" — and the model has no idea.

The pattern: **automatically re-attach a budgeted set of high-value content after compaction.**

The framework tracks throughout the session:

- The N most recently read files
- The N most recently used skills
- The currently-active plan (if any)
- The session memory entries

After compaction completes, the framework synthesizes a `post_compact` attachment containing:

```
POST_COMPACT_MAX_FILES_TO_RESTORE = 5    // top-5 recent files
POST_COMPACT_TOKEN_BUDGET = 50_000       // total budget for files
POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000 // per-file cap
POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000
POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000
```

These numbers are not arbitrary. They are tuned so the restoration consumes roughly half the freshly-cleared space, leaving the other half for continued work. Restoring too much defeats the compaction; restoring too little leaves the model amnesiac.

**Restoration content is structured as attachments.** Each file becomes a "system reminder" attachment that says "this file was recently read, here is its content (possibly truncated)." This is distinct from a fresh `FileRead` tool call — it does not consume a tool turn and the model knows this content is *background*, not action.

---

## Pattern 6: Prompt Cache Strategy

Prompt caching turns the static prefix into a near-free part of each request. The savings are real and large — for active sessions, 60–90% reduction in input token cost is achievable.

**Requirements for cache hits:**

- The first N bytes of the request must be byte-identical to the cached version
- "Identical" means: same model, same prompt text, same tools array (in the same order), same beta headers
- Cache TTL is 5 minutes from last hit (the cache is renewed each time it is read)

**Five common cache-breakers:**

1. **Date/time in the static prefix.** Daily cache invalidation across the entire user base.
2. **Tool order instability.** If the tools array is built by iterating an unordered map, the order varies and the cache misses.
3. **Feature flag flips mid-session.** A flag that switches the active branch in a section produces two prefix variants for the same user.
4. **Per-session env injected before the boundary.** CWD, git branch, env vars all break the cache if not in the dynamic section.
5. **MCP server connect/disconnect mid-session.** A late MCP connection adds tools, changing the tool list, breaking the cache for the rest of the session unless the tools were declared from the start.

**The cost-of-miss is asymmetric.** A cache miss reads the full prefix at full price. A cache hit reads it at ~10% of the price. So one accidental cache-breaker costs ~10× the savings of every hit, and you need many hits to recover. Audit aggressively.

---

## Pattern 7: Forks Share Their Parent's Cache

When a sub-agent forks from a parent, the goal is for both processes (parent and fork) to share the prefix cache. This requires byte-identical prefixes — system prompt and tools — between the two.

**The implementation detail that makes this work:** the fork inherits the parent's *rendered* system prompt bytes (passed via `renderedSystemPrompt`), not the parent's *configuration*. The parent's `getSystemPrompt()` is called once at fork-spawn time; the resulting string is threaded into the child's context; the child never calls `getSystemPrompt()` itself.

Why? Because the prompt depends on dynamic inputs (feature flag values, env info) that could resolve differently between the parent's spawn moment and the child's first API call:

- Feature flag with cold→warm transition
- Time-dependent values (date crosses midnight)
- Settings change between spawn and first call
- Memory file modified between spawn and first call

Any of these would produce a different prompt in the child, miss the parent's cache, and turn a "free" fork into a fully-priced one.

**The fork prompt also requires byte-identical tool_results.** All fork children inject placeholder tool_results with the literal string `"Fork started — processing in background"`. Only the final directive text differs per child. This pushes the cache boundary all the way to the directive.

The detailed fork mechanics are covered in `04-multi-agent.md`; the takeaway here is that **fork is a cache-sharing optimization, not a context-sharing primitive**. The context sharing is incidental; the cost savings is the point.

---

## Pattern 8: Tool Schema Token Economics

Tool schemas live in every request and add up fast. A typical schema is 200–700 tokens; an agent with 40 tools carries 8K–28K tokens of tools on every turn.

**The defer-and-discover pattern:**

- Core tools (used in 80%+ of turns) — always loaded
- Deferred tools — not in the request body; discoverable via a built-in `tool_search`
- `tool_search` returns the schemas of tools matching the model's query

The schema only enters the request when the model has called `tool_search` and got back a result. From the model's perspective, the toolset expanded mid-conversation; from the user's perspective, latency for `tool_search` (~100ms) is paid once for a tool the model needed, not 50 times for tools the model didn't.

**Quantifying.** If 30 of your 40 tools are domain-specific (not used most turns), deferring them saves roughly 30 × 400 = 12K tokens per turn. Across a million turns, that is 12B input tokens.

**The cost of deferral is one round-trip when the model needs a tool it doesn't have.** Sometimes this manifests as latency; sometimes the model proceeds without the tool because it didn't know to ask. Both costs are small relative to the savings, but tune the always-loaded set carefully.

---

## Pattern 9: Tool Result Budget

Tool results are the fastest-growing part of context. A search that returns 100K characters, a curl that downloads a JSON file, a build that produces verbose output — any of these can blow through several thousand tokens in one call.

**Two-level budget:**

- **Per-tool:** each tool declares `maxResultSizeChars`. The tool itself truncates, paginates, or offloads when its output exceeds this. The truncation strategy is the tool's choice (see `01-tool-design.md`, Pattern 5).
- **Per-conversation:** an aggregate budget across all tool results in the conversation. When the cumulative tool-result size exceeds the budget, the oldest results are evicted (replaced with `[Result evicted to save context]`).

The per-conversation budget is tracked in `contentReplacementState`, a per-conversation-thread structure that records:

- Which tool_use_ids have been replaced (so the replacement is idempotent across resends)
- Which tool_use_ids are still inert (already replaced, no need to evict again)
- The total size of remaining tool results

**Why a separate budget per thread?** Because sub-agents have their own conversation threads. The parent's budget tracks the parent's results; the sub-agent's tracks the sub-agent's. A budget shared across threads would create cross-thread eviction that surprises the model in one thread when something happened in another.

---

## Pattern 10: Disk Offload for High-Value Large Results

Some results are too large to keep in context but too valuable to discard. Examples:

- A scraped webpage with hundreds of KB of HTML
- A test report with thousands of test cases
- A build artifact list with all transitively included files
- A SQL query result with millions of rows

The pattern: persist the result to disk, return a reference, let the model selectively read pieces.

```python
if len(result) > MAX_INLINE_SIZE:
    path = persist_to_disk(result)
    return ToolResult(
        data=f"[Output too large ({len(result):,} chars). "
             f"Saved to: {path}. "
             f"Use FileRead with offset/limit to inspect sections.]"
    )
```

The model can then:

- `FileRead(path, offset=0, limit=200)` to see the start
- `Grep(pattern, path)` to find specific content
- `Glob` to enumerate related files

This converts a one-shot context burst into an addressable on-demand resource. Total cost is paid only for the pieces the model actually consumes.

**The offload directory has a lifecycle.** Per-session offloads live in a session-scoped tmp dir, cleaned up on session end. Persistent offloads (for cross-session reuse) live in a longer-lived path, garbage-collected by age and total size.

---

## Pattern 11: USD Budget as a Hard Cutoff

Autonomous and long-running agents need a financial circuit breaker. Token caps are not enough — token counts vary by model, and the relevant question is "how much money has this run cost?"

The pattern: `maxBudgetUsd` on the tool-use context. When the cumulative dollar cost of the run exceeds the budget, the framework refuses to make further API calls.

```python
class CostTracker:
    spent_usd: float
    budget_usd: float | None  # None = no budget cap

    def check_budget(self):
        if self.budget_usd is None:
            return  # no cap set
        if self.spent_usd >= self.budget_usd:
            raise BudgetExceeded(
                f"Spent ${self.spent_usd:.2f}, budget was ${self.budget_usd:.2f}. "
                f"Stopping. Increase --budget or review what consumed the budget."
            )
```

**The check fires *before* each API call**, so the budget excess is bounded by one call. Without this, an unattended agent could burn many multiples of the budget before noticing.

**Soft warnings precede the hard cap.** At 75% and 90% of budget, the framework injects a system message into the conversation telling the model how much it has left. The model can decide to wrap up. The hard cap is the last resort.

Detailed cost telemetry, including per-tool and per-stage attribution, is covered in `11-observability.md`.

---

## Pattern 12: Context Pressure Visible to the User

Token usage is opaque to users by default. The framework should expose:

- Current usage as a fraction of the effective ceiling
- A trend indicator (rising fast, stable, decreasing)
- The cost in dollars accumulated this session
- A "what would help?" hint when pressure is high (clear chat, compact, switch model)

This information should be in the UI footer, persistent, low-attention. Not a banner; a small indicator.

**Why this matters.** Users who can see the pressure adapt their workflow — they wrap up before forced compaction, they `/clear` between tasks, they spawn fresh subagents for distinct work instead of piling into one long conversation. Users who can't see the pressure get surprised when compaction fires mid-task.

---

## Token Economy Design Checklist

**Window calculation**
- [ ] Is the effective window the model's window minus output reserve minus safety buffer?
- [ ] Is the output reserve calibrated to p99 compaction summary size?
- [ ] Is there an env-var override for the auto-compact window?
- [ ] Are the three thresholds (warning, auto-compact, manual) defined separately with documented buffers?

**Compaction strategy**
- [ ] Is there an auto-compact path triggered before the wall, not at it?
- [ ] Is there a micro-compact for individual oversized tool results?
- [ ] Is there a snip-compact recovery for "prompt too long" deterministic failures?
- [ ] Is there a session-memory compact that persists extracted knowledge to disk?
- [ ] Are compact boundaries marked with structured metadata for resume?

**Post-compaction restoration**
- [ ] Is there a budgeted re-attachment of recent files after compaction?
- [ ] Is there a budgeted re-attachment of recent skills?
- [ ] Is the active plan re-attached if any?
- [ ] Are restoration content blocks marked as background reminders, not action requests?

**Cache hygiene**
- [ ] Is the system prompt prefix byte-stable across all sessions of the same agent type?
- [ ] Is the tools array order deterministic?
- [ ] Are mid-session feature flag effects on prompt content avoided?
- [ ] Are forks sharing the parent's prefix cache via threaded `renderedSystemPrompt`?
- [ ] Is there a tool that audits suspected cache misses (compare current prefix bytes to last cached)?

**Tool schemas**
- [ ] Are non-core tools deferred?
- [ ] Is `tool_search` available for runtime discovery?
- [ ] Do `searchHint` strings follow the formatting rules (3–10 words, no name overlap)?

**Tool results**
- [ ] Is `maxResultSizeChars` set on every tool?
- [ ] Is there a conversation-level result budget with eviction?
- [ ] Are oversized high-value results offloaded to disk with file references?
- [ ] Is offload storage GC'd by age and size?

**Cost governance**
- [ ] Is `maxBudgetUsd` available on long-running and autonomous runs?
- [ ] Are soft warnings emitted before the hard cap?
- [ ] Does the budget check fire *before* each API call?
- [ ] Is per-turn token usage visible to the user?

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Compact at the wall, not before | Compaction summary itself fails because there is no room | Trigger auto-compact at a buffered threshold |
| Single compaction strategy for everything | Bad fit for oversized tool results, bad fit for "prompt too long" recovery | Four strategies: auto, micro, snip, session-memory |
| Compact destroys all detail | User asks about a recently-read file and the model has no idea | Post-compact restoration of recent files, skills, plan |
| No `compact_boundary` marker | Resume replays pre-compact history, double-compaction cost | Structured boundary with metadata, durable in session storage |
| Forks re-render their own system prompt | Cache miss on every fork spawn | Thread `renderedSystemPrompt` from parent |
| Tools array ordered by dict insertion | Cache miss on Python or unordered-map iteration | Sort tools deterministically before serializing |
| Date in the static prefix | Daily cache invalidation across all users | Date in the env-info dynamic section |
| MCP servers connecting mid-session change the tool list | Late MCP connect breaks cache for the rest of session | MCP instructions in an uncached section, schemas added at session start where possible |
| All tools always-loaded | 10–25K wasted tokens per turn | Defer most tools, discover with `tool_search` |
| No per-conversation tool result budget | Context overflow after enough tool calls | Aggregate budget with oldest-first eviction |
| Large results inlined in context | One curl blows up the conversation | Disk offload with file reference |
| No `maxBudgetUsd` on autonomous runs | One bug burns through dollars before anyone notices | Hard cap with soft warnings before |
| User cannot see token pressure | Surprised by compactions mid-task | Visible footer with usage, cost, trend |
| Conversation-level budget shared across subagent threads | Cross-thread eviction surprises the model | Per-thread `contentReplacementState` |
| Snip-compact retried in a loop | Deterministic failure repeats forever | Detect repeat-with-same-input, fall back to a different strategy |
