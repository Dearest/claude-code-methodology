# Observability and Cost Governance Patterns

---

## The Core Insight

An agent without observability is a liability. Production agents make consequential choices, consume real money, and operate over long durations. The team running them needs to answer questions like:

- "What did the agent actually do this session?"
- "Why was tool X blocked? Why was Y auto-approved?"
- "Why is the bill 3× what it was last week?"
- "Which of our users are getting slow responses, and why?"
- "Which hook is the bottleneck?"
- "Did the autonomous run yesterday actually do anything useful, or just churn?"

A production observability surface answers all of these without forensic detective work. The pattern is: **emit structured events at every interesting moment, persist them in a queryable form, and surface them at three levels (real-time UI, session telemetry, long-term analytics).**

---

## Pattern 1: Cost Tracking as a First-Class State Type

Every API call costs money. Tracking it is not optional. The pattern: maintain a running cost-tracking state from session start, attribute every cost source, and expose the running total in the UI.

```typescript
type CostTracking = {
  totalUsd: number
  startedAt: timestamp

  // Per-source attribution
  byModel: Record<ModelId, ModelCosts>
  byTool: Record<ToolName, ToolCosts>
  byAgent: Record<AgentId, AgentCosts>     // for nested sub-agents
  byCategory: Record<CostCategory, number>  // 'input' | 'output' | 'cache_read' | 'cache_creation'

  // Detail per call
  recent: ApiCallCost[]                     // last N calls, for drill-down
}

type ModelCosts = {
  inputTokens: number
  outputTokens: number
  cacheReadTokens: number
  cacheCreationTokens: number
  usd: number
}
```

**Attribution dimensions worth tracking:**

- **Model** — Opus vs Sonnet vs Haiku have wildly different per-token cost
- **Tool** — which tools generated the most output (helps identify expensive tools)
- **Agent** — for nested topologies, which sub-agent burned through the budget
- **Category** — input vs output vs cache-read vs cache-creation; these have different prices
- **Stage** — primary model vs classifier vs compaction summary

Attribution makes the question "where did the cost go?" answerable. Without attribution, the answer is "I don't know."

---

## Pattern 2: USD Budget as Hard Cutoff

Cost tracking is monitoring. **Budget enforcement is action.** The pattern:

```python
class BudgetEnforcer:
    spent_usd: float
    budget_usd: float | None

    def check_before_api_call(self, estimated_cost):
        if self.budget_usd is None:
            return  # no cap
        projected = self.spent_usd + estimated_cost
        if projected > self.budget_usd:
            raise BudgetExceeded(
                f"Would spend ${projected:.2f}, budget was ${self.budget_usd:.2f}. "
                f"Stopping before this call."
            )

    def record_actual_cost(self, actual_cost):
        self.spent_usd += actual_cost
        self._maybe_emit_warning()

    def _maybe_emit_warning(self):
        if not self.budget_usd:
            return
        ratio = self.spent_usd / self.budget_usd
        if ratio > 0.9 and not self._warned_at_90:
            self._inject_system_warning("90% of budget consumed")
            self._warned_at_90 = True
        elif ratio > 0.75 and not self._warned_at_75:
            self._inject_system_warning("75% of budget consumed")
            self._warned_at_75 = True
```

**Three properties:**

1. **The check fires before the call, not after.** A post-call check can already be over budget. A pre-call check enforces the cap with at most one call's overshoot.

2. **Warnings precede the cap.** Soft warnings at 75% and 90% are injected into the model's context, letting the model wrap up gracefully. The hard cap is last-resort.

3. **The cap is opt-in.** Interactive sessions don't typically need a cap (the user sees the cost and can stop). Autonomous, scheduled, and remote sessions almost always do.

**The estimated cost is an approximation, not an oracle.** The framework estimates from token counts and current pricing; actual cost is reconciled after the response arrives. Discrepancies (extra cache writes, retries) are logged.

---

## Pattern 3: Per-Turn Performance Metrics

For each turn (user message → final assistant response), record:

```
turn_id
session_id
user_id (for analytics)
agent_type (if subagent)

start_ts
first_token_ts   → TTFT
end_ts           → wall-clock duration

api_calls         (count)
total_input_tokens
total_output_tokens
cache_read_tokens
cache_creation_tokens
total_usd

tool_uses
tool_use_types   (Record<name, count>)
tool_use_failures

interrupted: bool
compacted: bool
hit_budget_cap: bool

output_tokens_per_second  → OTPS
```

**TTFT and OTPS are user-perceived quality.** Users notice the difference between 800ms TTFT and 3s; between 100 OTPS and 30 OTPS. Tracking these surfaces performance regressions early.

**Per-turn data feeds session-level aggregates.** Sum over turns to get "this session cost $X, took Y minutes, used Z tools." Surface session aggregates in the UI footer.

---

## Pattern 4: Diagnostic Event Stream

Beyond metrics, the framework emits a structured event stream that captures *what happened*:

```typescript
type DiagnosticEvent =
  | { type: 'session_start', sessionId, model, ... }
  | { type: 'turn_start', turnId, sessionId, ... }
  | { type: 'api_call_start', callId, model, prompt_tokens, ... }
  | { type: 'api_call_end', callId, output_tokens, ttft_ms, ... }
  | { type: 'tool_use_start', toolUseId, tool, ... }
  | { type: 'tool_use_end', toolUseId, success, duration_ms, ... }
  | { type: 'permission_decision', toolUseId, behavior, reason, ... }
  | { type: 'classifier_call', stage, model, duration_ms, ... }
  | { type: 'hook_executed', hookName, event, exitCode, duration_ms, ... }
  | { type: 'compaction_triggered', reason, tokensBefore, tokensAfter, ... }
  | { type: 'subagent_spawned', parentId, childId, type, ... }
  | { type: 'task_created', taskId, type, ... }
  | { type: 'task_completed', taskId, status, ... }
  | { type: 'budget_warning', threshold, spent_usd, budget_usd }
  | { type: 'budget_exceeded', spent_usd, budget_usd }
  | { type: 'interrupt', reason }
  | { type: 'error', error_class, message, ... }
  // ...
```

**Each event is structured, typed, and indexed.** Querying "show me every permission decision today where the classifier was used" is a filter on `type` and `classifier_call`. Without structure, it's a `grep` over logs.

**Events are persisted, not just streamed.** A 24-hour-old event should be queryable. Persist to a local SQLite or log file by default; aggregate to a central service for fleet-wide analysis.

---

## Pattern 5: What to Log, What NOT to Log

**Always log:**

- Permission decisions with reason (audit)
- Tool calls with input summary (debugging)
- API call counts, tokens, costs (cost control)
- Errors with class and context (incident response)
- Compaction events (capacity planning)
- Subagent spawns with parent/child relationships (topology debugging)
- User interruptions (UX feedback)

**Never log:**

- File contents (privacy, size)
- Secrets, credentials, tokens (security)
- Raw user prompts at logging level beyond a hash (privacy unless explicit opt-in)
- Tool result contents in full (size; log summaries or sizes only)
- API call payloads in full (size + privacy)

**Log conditionally on operator policy:**

- Identifiers (user_id, session_id) — often required for support, but treat as PII
- Working directory paths — leak project structure
- Hostnames, machine identifiers — useful in fleet but PII in some contexts

The default should be safe: hash or summarize anything that could be sensitive; require explicit operator policy to log raw values.

---

## Pattern 6: Rate Limit Awareness

API providers rate-limit. A production agent must:

- Recognize rate-limit errors (HTTP 429, specific error messages)
- Implement exponential backoff with jitter
- Detect persistent rate-limiting and surface to the user
- Track rate-limit usage to predict approaching limits

The pattern: a rate-limit-aware retry policy:

```python
async def call_with_retry(request):
    for attempt in range(MAX_RETRIES):
        try:
            return await api.call(request)
        except RateLimitError as e:
            wait_seconds = compute_backoff(attempt, e.retry_after_seconds)
            await sleep(wait_seconds + jitter(wait_seconds))
        except DeterministicError:
            raise  # do not retry
    raise PersistentRateLimitError()
```

**Surface rate-limit status to the user.** When the agent is in the "waiting on rate limit" state, the UI should show it — "API rate-limited; retrying in 30 seconds." Without this, the user thinks the agent is hung.

**Track rate-limit headroom proactively.** If the API exposes "you have X requests remaining in this window," the framework can pace itself to avoid hitting the limit. This is much better than backing off after the fact.

---

## Pattern 7: Hook Latency Telemetry

User-installed hooks can be slow. A hook that adds 500ms to every tool call is invisible until users complain. The pattern: **per-hook latency tracking surfaced in user-visible telemetry.**

```typescript
type HookTelemetry = {
  hookName: string
  event: HookEvent
  calls: number
  totalDurationMs: number
  avgDurationMs: number
  p95DurationMs: number
  failures: number
  timeouts: number
}
```

**Surface the slowest hooks.** In the UI or via a `/hooks` command, show "Your hooks added 12s total this session. Slowest: `pre-commit.sh` (8s, 95th percentile)." The user can find and fix the offender.

**Auto-disable runaway hooks.** A hook that fails or times out repeatedly should auto-disable for the rest of the session, with a notification. The user can re-enable. Without this, one bad hook bricks the agent.

---

## Pattern 8: Subagent Cost Attribution

Sub-agents nest. A turn that spawns a coordinator that spawns 5 workers that each spawn 3 sub-workers can have hundreds of API calls. Attributing cost back to the original user turn requires structured chain tracking.

```typescript
type QueryChainTracking = {
  chainId: string   // shared across all descendants of one user turn
  depth: number     // 0 = main, 1 = direct child, 2 = grandchild, ...
}
```

**Every API call records its chain.** Telemetry aggregates by chain to answer "this turn cost $X total" (including all nested work).

**Show depth in the UI.** When a subagent's cost is large, the user wants to know "what spawned this?" — a depth indicator and parent chain shows the topology.

**Budget caps apply at the chain root, not per agent.** If the user sets a $5 budget, that's the cap for the entire chain — not $5 for the main agent, plus $5 per sub-agent. The cap belongs to the user's intent, which lives at the root.

---

## Pattern 9: Session Summary and Post-Run Audit

When a session ends, produce a structured summary:

```
Session: 2026-01-15-14:32-XYZ
Duration: 47 min
Cost: $4.27 (Opus: $3.91, Sonnet: $0.36)

Turns: 23
API calls: 89 (Opus: 31, Sonnet: 58)
Tokens: 1.2M input (87% cached), 142K output

Tools used:
  Bash:       45 calls, 12 with permission prompts
  FileRead:   31 calls
  FileEdit:   18 calls, 4 with permission prompts
  Agent:      3 spawns (1 worker, 2 verifier)

Tasks created: 5 (4 completed, 1 killed by user)
Compactions: 2 (auto, 47K → 18K)
Hook executions: 23 (avg 120ms)

Permission decisions: 16 (12 auto-approved by classifier, 4 user-confirmed, 0 denied)
Interruptions: 1 (user pressed escape during a slow Bash command)
Errors: 0
```

This summary is the answer to "what happened in this session?" — useful for the user post-session, for support cases, for cost analysis, for fleet-level analytics.

**Auto-archive sessions with summaries.** A directory of `session_id/summary.md` per session is a queryable history without needing a database. Operators can grep for "high-cost sessions this week" or "sessions where X was used."

---

## Pattern 10: Cost-Aware Decision Surfacing

The agent itself should be aware of its cost trajectory, not just measured. The pattern: inject cost context into the model's system reminders at decision points.

When the model is about to spawn a sub-agent:

```
<system-reminder>
You've spent $1.27 of $5.00 budget this session.
Last spawn (verifier agent) cost $0.43.
</system-reminder>
```

When approaching a budget threshold:

```
<system-reminder>
You've spent $3.85 of $5.00 (77%). Consider wrapping up before
the hard cap fires. Use the remaining budget for high-value work.
</system-reminder>
```

**The model should make cost-aware choices.** Knowing "I've spent X" lets the model:

- Choose a cheaper model when appropriate (downgrade to Sonnet from Opus for routine work)
- Avoid expensive operations near the cap (don't spawn another sub-agent at 90%)
- Prioritize finishing over exploring

Without cost visibility, the model treats every action as free.

---

## Pattern 11: OpenTelemetry-Style Distributed Tracing

For fleet-level deployments, distributed tracing is the right abstraction. Each agent run produces a trace:

```
Trace: session_xyz
  Span: turn_1
    Span: api_call_1 (model: Opus, 350ms)
    Span: tool_use_1 (Bash, 1200ms)
      Span: hook_pre_bash (200ms)
      Span: shell_exec (800ms)
      Span: hook_post_bash (180ms)
    Span: api_call_2 (model: Opus, 280ms)
    ...
```

Trace IDs let you join "this user reported a slow turn" to "exactly which calls were slow." Span attributes carry the structured event data described above.

**Trace exporters integrate with operator infrastructure.** Most teams already have OTLP-compatible backends (Honeycomb, Datadog, Grafana). The agent should export to these rather than reinventing.

---

## Pattern 12: The Three Audiences of Observability

A production observability system serves three audiences with different needs:

| Audience | Wants | Surface |
|---|---|---|
| **User in session** | What is the agent doing right now? How much have I spent? | UI footer, background task indicator, real-time progress |
| **User post-session** | What did the agent do this session? What did it cost? | Session summary, transcript with cost-per-turn |
| **Operator / SRE** | What's our fleet doing? Where are the slow paths, expensive paths, failing paths? | Distributed traces, aggregate metrics, alerts on anomalies |

A surface that serves only one audience leaves the other two blind. A coherent design covers all three with the same underlying event stream, differently aggregated and filtered.

---

## Observability Design Checklist

**Cost tracking**
- [ ] Is cost tracked per session as part of AppState?
- [ ] Are costs attributed across model, tool, agent, category, stage?
- [ ] Are recent calls preserved for drill-down?
- [ ] Is the running total surfaced in the UI footer?

**Budget enforcement**
- [ ] Is `maxBudgetUsd` available for autonomous and scheduled runs?
- [ ] Does the check fire *before* each API call?
- [ ] Are soft warnings injected at 75% and 90% thresholds?
- [ ] Is the hard cap last-resort, with the model given a chance to wrap up first?

**Per-turn metrics**
- [ ] Are TTFT and OTPS tracked per turn?
- [ ] Are token counts (input, output, cache read, cache creation) recorded?
- [ ] Are tool use counts and types recorded?

**Event stream**
- [ ] Is there a structured, typed event stream covering all the key moments?
- [ ] Are events persisted (SQLite, log files), not just emitted to stdout?
- [ ] Are events queryable by type, time, session, user, agent?

**Logging policy**
- [ ] Are permission decisions, tool calls, costs always logged?
- [ ] Are file contents, secrets, raw user prompts never logged?
- [ ] Are identifiers (user_id, session_id, paths) gated on operator policy?

**Rate limits**
- [ ] Does the framework recognize rate-limit responses and back off with jitter?
- [ ] Is the rate-limit waiting state visible in the UI?
- [ ] Does the framework track headroom proactively if the API exposes it?

**Hook telemetry**
- [ ] Are per-hook latencies tracked?
- [ ] Are slow hooks surfaced via `/hooks` or equivalent?
- [ ] Do runaway hooks auto-disable for the session?

**Subagent attribution**
- [ ] Does every API call carry a chain ID rooted at the user turn?
- [ ] Are budget caps applied at the chain root, not per agent?
- [ ] Does the UI show subagent topology and depth?

**Session summary**
- [ ] Is a structured summary produced at session end?
- [ ] Are sessions auto-archived to a queryable directory or store?

**Cost-aware decisions**
- [ ] Does the model see its cost trajectory via system reminders at key moments?
- [ ] Are warnings near the cap part of the prompt, not just the UI?

**Distributed tracing**
- [ ] For fleet deployments, is there a trace exporter for OTLP or equivalent?
- [ ] Do traces preserve the session/turn/api-call hierarchy?

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Cost only logged, not tracked in-session | User has no idea what's been spent | Running total in AppState; UI footer |
| Single cost number, no attribution | Can't answer "where did the money go" | Per-model, per-tool, per-agent breakdown |
| Budget enforced via post-call check | Easy to overshoot by one expensive call | Pre-call check with estimated cost |
| Budget cap with no warnings | Hard cap surprises both user and model | Soft warnings at 75% and 90% via system reminders |
| TTFT and OTPS not tracked | Performance regressions invisible | Per-turn metrics from API timestamps |
| Plain-text logs with structured fields buried in strings | Querying requires grep + regex | Structured event types with typed fields |
| Logging file contents | Privacy violation + log size explosion | Log summaries (size, line count, file path), not contents |
| Logging raw API payloads | Privacy + size + can include secrets | Log summary metadata only |
| Rate limit handling as plain retry | Tight retry loops worsen the limit | Exponential backoff with jitter, surface waiting state |
| Hook latency invisible | Slow hooks degrade UX silently | Per-hook latency surfaced via UI |
| Hook failures bring down the agent | One bad hook bricks the session | Auto-disable on repeated failure, notify |
| Subagent costs charged to subagent only | User sees subagent cost without context | Chain-rooted attribution; budget at chain root |
| No session summary | Post-session audit requires reading entire transcript | Structured summary at session end |
| Cost hidden from the model | Model treats every operation as free | Cost system reminders at decision points |
| Observability surfaces only for ops, not users | Users blind to what their agent is doing | Three audiences, three surfaces, one event stream |
| Telemetry exported to a custom format | Operator already has Datadog/Honeycomb/Grafana; doesn't want to integrate yet another | OTLP-compatible exporter |
