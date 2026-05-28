# Conversation Loop and Query Engine Patterns

---

## The Core Insight

The conversation loop is the agent's heart. Every other dimension (tools, memory, permissions, compaction) services the loop. A robust loop is what separates an agent that survives a 12-hour migration from one that crashes after three turns.

The loop has a deceptively simple top-level shape:

```
while not done:
    response = call_model(messages, tools, system_prompt)
    for each block in response.content:
        if block.type == 'text':
            stream block to UI
        if block.type == 'tool_use':
            schedule tool execution
    wait for all tools to complete
    if no tool_use blocks (stop_reason == 'end_turn'):
        break
    messages.append(response)
    messages.append(tool_results)
```

The complexity is in the *also-true*: streaming, interruption, parallel tool execution, partial responses, retry, compaction triggers, orphaned tool calls, missing results, withheld output, async tool injection of new messages, sub-agent context derivation, and budget enforcement. Each of these is a real production concern.

---

## Pattern 1: The Async Generator as Loop Spine

The loop is implemented as an async generator that yields messages as they become available:

```typescript
async function* query(params): AsyncGenerator<Message> {
    while (!done) {
        for await (const message of callModel({...})) {
            yield message            // model output streams to consumer
            // process tool_use blocks, append to history
        }
        if (stop_reason === 'end_turn') done = true
    }
}
```

This shape gives three things at once:

1. **Streaming.** Each model chunk yields to the consumer immediately — the UI displays text as it's generated, not all at the end.
2. **Backpressure.** If the consumer is slow, the generator naturally suspends. No buffer overflow, no dropped messages.
3. **Cancellation.** The consumer can stop iterating; the generator's `finally` blocks unwind. Resources released cleanly.

The consumer (the REPL, the SDK, the server) iterates the generator without needing to know the loop's internal details.

---

## Pattern 2: Turn Boundaries and Stop Reasons

Every model response carries a `stop_reason`. The loop's behavior depends on which one:

| stop_reason | Meaning | Loop response |
|---|---|---|
| `end_turn` | The model finished its message naturally | Break the loop; turn complete |
| `tool_use` | The model wants to call tools | Execute tools, loop with results |
| `max_tokens` | The model hit its output budget | Detect via the `withheld_max_output_tokens` heuristic; either continue silently or surface |
| `pause_turn` | Mid-turn pause (rare; SDK-driven) | Yield control, resume on next continue |
| `stop_sequence` | A configured stop sequence fired | Treat like `end_turn` |
| `error` | API error | Retry with backoff or surface |

**The `withheld_max_output_tokens` heuristic** detects when the model wanted to keep talking but was cut off mid-output. The signal is: `stop_reason == 'max_tokens'` AND there's an unclosed tool_use block OR text ending mid-sentence OR no signs of natural completion. When this fires, the loop offers the model a chance to continue — typically by re-prompting with "continue" and concatenating the result.

**One turn can contain multiple model calls.** A single "user message → assistant turn" boundary often wraps several `call_model` invocations: the model uses tools, gets results, uses more tools, gets more results, eventually says `end_turn`. Code that conflates "turn" with "API call" misses this.

---

## Pattern 3: Parallel and Concurrency-Safe Tool Execution

When an assistant response contains multiple `tool_use` blocks, the loop can execute them in parallel — but only if every tool involved declares itself concurrency-safe for the given inputs.

```python
# Group concurrent-safe tools, serialize the rest
concurrent_safe = [t for t in tool_calls if t.tool.is_concurrency_safe(t.input)]
serial = [t for t in tool_calls if not t.tool.is_concurrency_safe(t.input)]

# Run safe tools in parallel
parallel_results = await asyncio.gather(*[run(t) for t in concurrent_safe])

# Run unsafe tools in sequence
serial_results = []
for t in serial:
    serial_results.append(await run(t))
```

**Why this matters.** A single assistant turn often issues many parallel reads — read these 5 files, search for X, list files in Y. Running them serially adds 5× latency for no benefit. Running them parallel adds latency benefit and no correctness cost (reads don't interfere).

**Concurrent execution must preserve result order.** The model expects tool results in the same order as the tool_use blocks. The framework gathers parallel results, then orders them by the original index before injecting into history.

**`contextModifier` forces serialization.** A tool that returns a `contextModifier` cannot run in parallel with siblings (because the modified context would race). The framework checks this and forces the tool into the serial group.

---

## Pattern 4: Streaming Tool Execution

For long-running tools (multi-second shell commands, large file processing, web fetches), waiting for the tool to finish before showing anything is a bad UX. The pattern: **streaming tool execution with progress events.**

```python
async def run_tool_streaming(tool, input, context):
    progress_callback = lambda data: yield_progress(tool_use_id, data)
    result = await tool.call(input, context, on_progress=progress_callback)
    yield ToolResult(tool_use_id, result.data)
```

The tool's progress callback emits `ToolProgress` messages that the consumer renders in real time — a Bash tool streams its stdout, a fetch tool reports bytes received, a search tool reports matches found.

**Progress data has a structured type.** Each progress event is typed (`BashProgress`, `MCPProgress`, `WebSearchProgress`, etc.) so the UI knows how to render it. Generic progress (just a string) is fine for simple cases; rich progress (structured fields, percentage, item count) gives better UI affordances.

**Progress messages do not enter the model's context.** They are UI-only. The model sees only the final tool result. Without this separation, a streaming tool would inflate the model's context with progress noise.

---

## Pattern 5: Cancellation Through AbortController

The loop must support interruption — the user clicks "stop", the budget cap fires, the parent agent kills a worker. The pattern: a shared `AbortController` in the tool-use context, checked at every async boundary.

```python
class ToolUseContext:
    abort_controller: AbortController
    # ...

# In the loop:
if context.abort_controller.signal.aborted:
    # Cleanup, finalize, exit
    break

# In a tool:
def call(self, input, context):
    for chunk in stream:
        if context.abort_controller.signal.aborted:
            raise Cancelled()
        process(chunk)
```

**Three categories of interruptible work:**

1. **Model calls.** The HTTP request to the API is abortable; the framework cancels in-flight calls when the abort fires.
2. **Tool execution.** Tools check the signal at loop boundaries. A long-running tool that ignores the signal becomes effectively un-interruptible.
3. **Post-tool cleanup.** Tools mid-execution at abort time get the chance to clean up (close files, kill subprocesses) before the loop unwinds.

**Abort reasons matter.** The `signal.reason` field distinguishes:

- `'interrupt'` — user-initiated, show "interrupted" message in transcript
- `'submit-interrupt'` — user sent a new message; suppress the "interrupted" indicator (the new message takes its place)
- `'budget'` — budget exceeded; emit the budget message
- `'timeout'` — wall-clock or per-tool timeout

Each reason gets different post-cancellation handling.

---

## Pattern 6: Orphaned Tool Calls and Missing Result Blocks

The API requires that every `tool_use` block be followed by a `tool_result` block with the same `tool_use_id`. Mismatches produce an API error.

This invariant is fragile: a tool that crashes, an interrupt mid-execution, an attempt to resume a partially-completed turn, a malformed tool result — any of these can leave the message history with orphaned `tool_use` blocks (no matching results) or orphaned `tool_result` blocks (no matching uses).

**The framework checks and repairs at every loop iteration.** Before sending the next API request, it scans the message history for:

- `tool_use` blocks with no matching `tool_result` → inject a synthetic `tool_result` with content like "Tool execution was interrupted." Keep the conversation valid.
- `tool_result` blocks with no matching `tool_use` → drop the orphan; it has no semantic meaning.

The synthetic results are the smaller surface area than the alternatives (dropping the assistant message, or refusing to continue). The model handles "interrupted" gracefully — it sees the partial work and adapts.

---

## Pattern 7: The Compact Boundary System Message

When compaction fires, the loop must emit a structured marker so:

- Future iterations know to load the post-compact state, not the pre-compact history
- Resume code knows where to truncate when restoring a session
- SDK consumers know that prior content is summarized, not in context

The marker is a `system` message with `subtype: 'compact_boundary'` and metadata:

```typescript
{
  type: 'system',
  subtype: 'compact_boundary',
  compactMetadata: {
    preservedSegment?: { tailUuid: string },
    summaryText: string,
    triggerReason: 'auto' | 'manual' | 'micro' | 'session_memory',
    tokensBeforeCompact: number,
    tokensAfterCompact: number,
  }
}
```

The compaction implementation is in `05-token-economy.md`; the loop's responsibility is *recognizing and respecting* the boundary:

- The loop iterates message history and acknowledges `compact_boundary` messages immediately (they appear immediately after the user, ahead of the model's response).
- When yielding to SDK consumers, the boundary is yielded as its own message, distinct from assistant or user content.
- Pre-boundary messages are released for GC after the boundary is committed — they will not be needed again in this session.

**Pre-compaction flush.** Before writing the boundary, the loop flushes any in-memory-only state (the file state cache's hash table, the cost-tracking entries) to durable storage. Without this, a crash mid-compaction would lose state that should have survived.

---

## Pattern 8: Sub-Agent Spawning Inside the Loop

When the assistant calls an agent-spawning tool, the loop's behavior depends on whether the spawn is inline or background:

**Inline spawn.** The loop yields control to the spawn, awaits the child's full result, then resumes. The child's result becomes the tool result for the spawning tool_use block. The child's turns do not appear in the parent's transcript — only the final summary does.

**Background spawn.** The loop returns immediately with a task ID as the tool result. The child runs asynchronously. The child's eventual result arrives later via the `<task-notification>` envelope as a synthetic user message in the parent's next turn.

**Query chain tracking.** Every spawn records its parent-child relationship in a chain ID:

```typescript
type QueryChainTracking = {
  chainId: string  // shared root for all descendants of one user turn
  depth: number    // how many spawns deep we are
}
```

The chain ID lets analytics group all model calls that derive from a single user message. Without it, a turn that spawned 5 sub-agents looks like 6 unrelated conversations in the metrics.

---

## Pattern 9: Retry, Backoff, and Deterministic Failure Detection

API calls fail. The loop's retry policy is layered:

| Failure category | Detection | Response |
|---|---|---|
| **Transient** (429 rate limit, 503, network timeout) | Status code | Exponential backoff with jitter; retry |
| **Deterministic** (prompt too long, invalid input) | Status code + error message | Do not retry; fall back to a different strategy |
| **Authentication** (401, 403) | Status code | Refresh credentials; if still failing, surface to user |
| **Server-side** (500 with no specifics) | Status code | Retry once; surface if persists |

**Deterministic failures are the dangerous class.** Retrying a deterministic failure with the same input fails the same way. The classic example is "prompt too long" — the retry hits the same wall every time, burning latency and money. The loop must recognize the deterministic case and switch strategies (snip-compact, request fewer tokens, drop attachments) instead of looping.

**Retry budget is a property of the turn, not the call.** A single turn can retry up to N times across all its API calls. Beyond N, the turn fails and surfaces an error. This prevents one bad model state from consuming an unbounded amount of dollars in retries.

**Retry preserves cache hits.** A retry sends the same request, so the prompt cache hits. The cost of a retry is one extra request, not one extra full prefix.

---

## Pattern 10: Withheld Output Token Detection

When the model is cut off mid-stream (it ran out of output budget), the chunked stream may not signal this clearly. The pattern: a heuristic that fires when stop_reason is `max_tokens` AND certain content-shape signals are present:

- Unclosed tool_use block (the model started a tool call and the closing structure was withheld)
- Text ending mid-token (no terminal punctuation, no newline)
- Text ending mid-XML-tag
- Assistant content shorter than the expected response shape

When the heuristic fires, the loop offers continuation:

```
[assistant content cut off at max_tokens]

[user]: continue

[assistant continues seamlessly]
```

The continuation is automatic in interactive mode (the user sees only the combined response), surfaced as a choice in headless mode (the user gets the partial response and can ask for more).

**False positives are okay; false negatives are not.** A false positive triggers one extra harmless model call. A false negative leaves the user with a truncated response and no recovery path.

---

## Pattern 11: Hook Integration in the Loop

Hooks fire at well-defined points in the loop. The loop's responsibility is invoking the hook handler at the right moment and respecting its result:

```python
async def call_tool_with_hooks(tool, input, context):
    # PreToolUse — can block, can modify input
    pre_result = await run_hooks('PreToolUse', tool, input)
    if pre_result.blocked:
        return synthesize_blocked_result(pre_result.reason)
    if pre_result.modified_input:
        input = pre_result.modified_input

    # Actual tool call
    try:
        result = await tool.call(input, context)
        # PostToolUse — observe only
        await run_hooks('PostToolUse', tool, input, result)
        return result
    except Exception as e:
        # PostToolUseFailure — observe failure
        await run_hooks('PostToolUseFailure', tool, input, e)
        raise
```

**Hook latency adds up.** Five hooks at 200ms each = 1 second added to every tool call. The loop should:

- Run hooks in parallel where ordering doesn't matter
- Time out hooks at a generous but finite limit (5–30 s)
- Surface slow hooks in the UI so users can find the culprit

**Hook failures are observable.** A hook that exits with an error doesn't crash the loop; it's recorded with `outcome: 'error'` and the loop proceeds. The audit trail tells the user why.

---

## Pattern 12: Per-Turn API Metrics

The loop tracks per-turn metrics that drive both real-time UI and long-term observability:

- **Time to first token (TTFT)** — model latency before the first byte of response
- **Output tokens per second (OTPS)** — sustained throughput
- **Cumulative tokens** — input + output across the turn
- **Cumulative cost** — converted to USD using current pricing
- **Tool count** — how many tool calls executed
- **API call count** — how many round-trips
- **Wall-clock duration** — start to finish

These metrics are emitted at end-of-turn for telemetry and also surfaced in the UI footer for the user.

**TTFT and OTPS are the user-visible quality bar.** Users notice the difference between 800ms TTFT and 3s TTFT; they notice the difference between 100 OTPS and 30 OTPS. Tracking and surfacing these makes performance regressions visible early.

Full observability and cost governance are covered in `11-observability.md`.

---

## Conversation Loop Design Checklist

**Loop structure**
- [ ] Is the loop an async generator that yields messages as they stream?
- [ ] Does it handle stop_reasons explicitly (end_turn, tool_use, max_tokens, others)?
- [ ] Does it distinguish "turn" from "API call" — one turn can have many calls?
- [ ] Is there a top-level `done` condition that explicitly evaluates after each iteration?

**Tool execution**
- [ ] Are concurrency-safe tools run in parallel, others serialized?
- [ ] Are tool results returned in the original tool_use order?
- [ ] Do streaming tools emit `ToolProgress` events that the UI can render?
- [ ] Are progress events distinct from final results and excluded from model context?

**Cancellation**
- [ ] Is there a shared `AbortController` in the tool-use context?
- [ ] Does every async boundary check the abort signal?
- [ ] Are abort reasons (`interrupt`, `submit-interrupt`, `budget`, `timeout`) distinguishable?
- [ ] Do tools have cleanup hooks for mid-execution abort?

**Invariants**
- [ ] Before every API call, are orphaned `tool_use` blocks repaired with synthetic results?
- [ ] Are orphaned `tool_result` blocks dropped?
- [ ] Is the message history valid before sending?

**Compaction integration**
- [ ] Does the loop emit a structured `compact_boundary` marker after compaction?
- [ ] Does the loop release pre-compact messages for GC after the boundary?
- [ ] Does pre-compaction flush in-memory state to durable storage?

**Sub-agent integration**
- [ ] Are inline and background spawns distinguishable in the loop?
- [ ] Is `QueryChainTracking` set with chainId and depth for every spawn?
- [ ] Do task-notification messages enter the parent's history as synthetic user messages?

**Resilience**
- [ ] Are retries categorized by transient/deterministic/auth/server?
- [ ] Are deterministic failures (prompt too long) handled with strategy switch, not loop?
- [ ] Is there a per-turn retry budget?
- [ ] Are retries cache-friendly (preserve the cache hit)?

**Output handling**
- [ ] Is there a withheld-max-output heuristic that detects cutoff?
- [ ] Does continuation happen automatically in interactive, prompted in headless?

**Hook integration**
- [ ] Do PreToolUse hooks see input and can return a modified version?
- [ ] Do PostToolUse hooks observe results?
- [ ] Do hooks have a generous but finite timeout?
- [ ] Do hook failures degrade gracefully?

**Metrics**
- [ ] Does the loop track TTFT, OTPS, token usage, cost?
- [ ] Are metrics emitted at end-of-turn for telemetry?

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Loop as plain async function (no generator) | No streaming, no backpressure, all-or-nothing | Async generator with yield-per-chunk |
| Serial tool execution always | 5× latency for parallel-safe operations | Group by concurrency-safety flag |
| Tool results returned out of order | Model sees mismatched order, gets confused | Preserve original tool_use index |
| Progress events enter the model's context | Token inflation; model sees noise | UI-only stream separate from message history |
| No abort signal in tool context | Tools can't be interrupted; users wait | Shared `AbortController`, check at boundaries |
| Single "aborted" reason | Can't distinguish user-stop from budget-stop from new-message | Tagged abort reasons with different handling |
| No orphan repair | API errors on resume or after interrupt | Synthesize missing tool_results before next API call |
| No `compact_boundary` marker | Resume replays pre-compact history; double compaction | Structured boundary message with metadata |
| Compaction without pre-flush of in-memory state | Crash mid-compact loses state that should survive | Flush before writing boundary |
| Retry deterministic failures | Loop forever, burn money | Detect deterministic, switch strategy |
| No retry budget | One bad turn consumes unbounded cost | Per-turn retry cap |
| No withheld-output detection | Truncated responses delivered with no recovery | Heuristic for `max_tokens` + content-shape |
| Hooks block the loop forever | Agent hangs invisibly | Per-hook timeout |
| Loop doesn't track query chain | Telemetry can't group calls by user turn | `QueryChainTracking` with chainId, depth |
| Treating every API call as a separate turn | Misattributes work, breaks per-turn budgets | Turn = user → assistant boundary, can span many calls |
| Sub-agent results appended as assistant messages | History looks like the parent made the sub-agent's findings | Synthetic user message with `<task-notification>` envelope |
