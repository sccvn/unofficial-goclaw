# ADR-002: 8-Stage Agent Pipeline (Chain of Responsibility)

**Status:** Accepted  
**Date:** 2025-02-01  

---

## Context

The AI agent loop must perform several ordered operations before and after calling an LLM:

1. **Load context files** (per-agent and per-user) into system prompt
2. **Truncate history** to stay within token budget
3. **Build the prompt** (system prompt + history + new message)
4. **Call the LLM** (streaming or blocking)
5. **Execute tool calls** from the LLM response, potentially re-entering the loop
6. **Observe results** — collect outputs, handle errors, re-inject tool results
7. **Flush memory** — write episodic summaries, trigger consolidation
8. **Summarize** — produce the final response, trim history

These operations are interdependent in sequence but must be independently testable, replaceable, and extensible via callbacks.

### Forces

- Different LLM providers have different token limits and cost structures — context management must be pluggable.
- The "act" stage is recursive: tool calls can trigger sub-stages or even spawn sub-agents.
- Operators want to inject custom behavior (webhooks, logging, guardrails) at specific stages without forking the core loop.
- Unit testing individual stages in isolation requires clean dependency injection.
- Pipeline state (messages in flight, tool results, token counts) must flow between stages without global mutation.

---

## Decision

Implement the pipeline as an **8-stage Chain of Responsibility** (`internal/pipeline/`) with a shared mutable `RunState` passed through each stage sequentially.

```
context → history → prompt → think → act → observe → memory → summarize
```

### Stage Contract

```go
// internal/pipeline/stage.go
type Stage interface {
    Run(ctx context.Context, state *RunState) error
}
```

Each stage reads from and writes to `RunState`. Any stage may return an error to abort the chain. The `think` stage (LLM call) populates `state.Response`; subsequent stages consume it.

### RunState (key fields)

```go
type RunState struct {
    AgentID        uuid.UUID
    SessionKey     string
    Messages       []store.Message  // mutable — truncated in history stage
    SystemPrompt   string           // built in prompt stage
    Response       *providers.Response
    ToolResults    []store.ToolResult
    TokensUsed     int
    ShouldContinue bool             // false → stop after this turn
    // ...callbacks, tracing, substate
}
```

### Callback Hooks

Stages fire lifecycle callbacks via `RegisterCallback()` at well-defined points (pre-think, post-think, pre-tool, post-tool, post-turn). This enables:
- LLM call tracing (write trace spans)
- Channel streaming (push partial tokens to WebSocket)
- Hook dispatcher (pre/post tool use blocking events)
- Custom telemetry

```go
pipeline.RegisterCallback(pipeline.EventPreThink, func(ctx context.Context, s *RunState) {
    tracing.StartLLMSpan(ctx, s.AgentID, s.Messages)
})
```

### Always-On Execution Path

The `think` → `act` → `observe` stages form a **recursive loop** when tools are called. The `act` stage inspects `state.Response.ToolCalls`, executes each tool, and re-invokes `think` with tool results appended. This loop continues until the LLM returns no more tool calls or `MaxToolRounds` is reached.

```
think ──(tool calls?)──► act ──► observe ──► think (recurse)
                 └──(no calls)──► memory ──► summarize
```

---

## Consequences

### Positive

- Each stage is independently testable with mocked `RunState`.
- New capabilities (e.g., a new pruning algorithm) require only a replacement `Stage` implementation.
- Callbacks decouple cross-cutting concerns (tracing, streaming, hooks) from pipeline logic.
- The recursive `think → act` loop naturally handles multi-step tool usage without extra orchestration.

### Negative

- `RunState` grows over time as stages add fields — discipline required to not make it a grab-bag.
- The recursive `act → think` loop requires careful depth/round limits to prevent infinite recursion.
- Stage execution is sequential — there is no parallel stage execution. Parallelism happens at the sub-agent level, not the pipeline stage level.
- Callback registration order matters; mis-ordered callbacks can produce incorrect traces.

### Risk Mitigations

- `MaxToolRounds` constant prevents infinite tool loops.
- `RunState` fields are documented with which stage writes them and which stages read them.
- Integration tests in `internal/pipeline/` run the full chain with a mock provider.

---

## Alternatives Considered

| Alternative | Rejected Because |
|-------------|-----------------|
| Monolithic function | Untestable; impossible to inject callbacks at specific points |
| Middleware stack (HTTP-style) | Natural for request/response; awkward for the recursive think→act loop |
| Event-driven stages (async) | Adds latency and complexity; ordering guarantees are harder to enforce |
| Explicit state machine | Overkill for a linear + one-loop flow; adds boilerplate without gain |
