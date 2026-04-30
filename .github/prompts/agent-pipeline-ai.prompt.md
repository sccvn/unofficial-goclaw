---
description: 'AI agent pipeline development for GoClaw: 8-stage pipeline (context→history→prompt→think→act→observe→memory→summarize), LLM provider adapters, tool execution, memory consolidation, orchestration patterns. Use when extending the pipeline or provider layer.'
---

# AI Agent Pipeline — Development Prompt

## 8-Stage Pipeline Overview

Defined in `internal/pipeline/pipeline.go`. Stages run sequentially; each receives and mutates `*RunState`.

```
Stage 1: context     — load agent context files, user context files
Stage 2: history     — load conversation history (L0/L1/L2 memory tiers)
Stage 3: prompt      — assemble system prompt (SOUL.md + IDENTITY.md + context)
Stage 4: think       — call LLM, collect streaming response
Stage 5: act         — execute tools returned by LLM
Stage 6: observe     — collect tool results, re-enter think if needed
Stage 7: memory      — flush working memory, trigger episodic consolidation
Stage 8: summarize   — finalize session, update stats, trigger post-turn hooks
```

## Stage Implementation Contract

```go
// internal/pipeline/stage.go
type Stage interface {
    Name() string
    Run(ctx context.Context, state *RunState) error
}

// Abort check pattern — respect upstream abort
func (s *MyStage) Run(ctx context.Context, state *RunState) error {
    if state.IsAborted() || ctx.Err() != nil {
        return nil
    }
    // stage logic...
    return nil
}
```

## RunState Key Fields

```go
// internal/pipeline/run_state.go
type RunState struct {
    Agent       *store.Agent
    Session     *store.Session
    UserMessage string
    Messages    []Message          // conversation history
    SystemPrompt string            // assembled in prompt stage
    Response    *LLMResponse       // filled in think stage
    ToolCalls   []ToolCall         // filled in think stage
    ToolResults []ToolResult       // filled in act/observe stage
    MemoryItems []MemoryItem       // for memory flush stage
    // ... abort, metadata, etc.
}
```

## 3-Tier Memory Architecture

```
L0 — Working memory (current conversation messages) — always loaded
L1 — Episodic memory (session summaries) — loaded on demand
L2 — Semantic memory (KG entities + pgvector embeddings) — loaded on demand

Progressive loading: L0 auto-injected → L1 loaded if context budget allows → L2 similarity search
```

## Provider Adapter Pattern

```go
// internal/providerresolve/ — ProviderAdapter interface
type ProviderAdapter interface {
    Complete(ctx context.Context, req CompletionRequest) (*CompletionResponse, error)
    Stream(ctx context.Context, req CompletionRequest, handler StreamHandler) error
    ModelInfo(modelID string) (*ModelInfo, bool)
}

// Streaming — reuse SSEScanner from internal/providers/sse_reader.go
scanner := NewSSEScanner(resp.Body)
for scanner.Scan() {
    event := scanner.Event()
    // process event...
}

// Retries — always wrap HTTP calls
result, err := RetryDo(ctx, 3, func() (*Response, error) {
    return client.Do(req)
})
```

## Tool Execution Pattern

Tools are registered in `internal/tools/`. Each tool implements:

```go
type Tool interface {
    Name() string
    Description() string
    Schema() json.RawMessage  // JSON Schema for LLM function calling
    Execute(ctx context.Context, params json.RawMessage) (string, error)
}
```

Tool gating for Lite edition (`internal/tools/team_action_policy.go`):
```go
// Lite blocks: comment, review, approve, reject, attach, ask_user
// skill_manage and publish_skill are not registered in Lite
```

## Agent Identity Convention

| Field | Use for |
|---|---|
| `agent.ID` (UUID) | DB foreign keys, event payloads, internal routing |
| `agent.AgentKey` (string slug) | Log messages, file paths, UI labels, workspace dirs |

**Never** use `agent_key` as a DB foreign key. **Never** use UUID in workspace file paths.

## Memory Consolidation Workers

Defined in `internal/consolidation/`. Three workers:
1. **Episodic** — summarizes sessions into episodic memory
2. **Semantic** — extracts entities into knowledge graph
3. **Dreaming** — background consolidation of episodic → semantic during idle

Workers are triggered via `DomainEventBus` in `internal/eventbus/`. Events use worker pool with dedup + retry.

## Orchestration Patterns

### BatchQueue[T] — Aggregate Sub-agent Results
```go
// internal/orchestration/
q := orchestration.NewBatchQueue[ChildResult](numWorkers)
q.Submit(task1)
q.Submit(task2)
results := q.Collect()
```

### Delegate Tool — Inter-agent Delegation
Three modes: `auto` (system chooses), `explicit` (caller specifies agent), `manual` (human-in-loop).
Configured via `agent_links` table. Token-aware work distribution.
