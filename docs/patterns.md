# GoClaw — Design Patterns Catalogue

Inventory of recurring patterns implemented in this codebase.

---

## 1. Interface-Based Store Abstraction (Strategy Pattern)

**Location:** `internal/store/stores.go`, `internal/store/pg/`, `internal/store/sqlitestore/`

Each store domain (agents, sessions, memory, vault, etc.) is defined as a Go interface. Both PostgreSQL and SQLite provide independent implementations. The gateway is initialized with one or the other based on build tags.

```go
// Interface: internal/store/stores.go
type AgentStore interface {
    GetAgent(ctx context.Context, id uuid.UUID) (*Agent, error)
    // ...
}

// Two concrete implementations:
// internal/store/pg/agent_store.go  — PostgreSQL
// internal/store/sqlitestore/agent_store.go — SQLite
```

**Benefit:** The pipeline and HTTP handlers are DB-agnostic. Desktop edition (`sqliteonly`) substitutes SQLite transparently.

---

## 2. Pipeline Stage Chain (Chain of Responsibility)

**Location:** `internal/pipeline/`

The 8-stage agent pipeline runs each stage sequentially, passing a shared `RunState`. Each stage reads and mutates state; any stage can signal abort.

```
context → history → prompt → think → act → observe → memory → summarize
```

Each stage implements `Stage.Run(ctx, state)`. Stages are composable and testable in isolation. Pluggable via `RegisterCallback()` hooks.

---

## 3. Provider Adapter (Adapter Pattern)

**Location:** `internal/providers/`, `internal/providerresolve/`

All LLM providers implement `ProviderAdapter`. A `ModelRegistry` maps model IDs to provider instances with forward-compatibility resolution. This allows routing `gpt-4o` → OpenAI-compat provider, `claude-3-5-sonnet` → Anthropic provider, etc.

```go
type ProviderAdapter interface {
    Complete(ctx, req) (*Response, error)
    Stream(ctx, req, handler) error
    ModelInfo(modelID string) (*ModelInfo, bool)
}
```

---

## 4. Context Propagation (Context Object Pattern)

**Location:** `internal/store/context.go`

Tenant, user, agent, locale, and agent-type values are threaded through `context.Context` using typed keys. Downstream store implementations extract these values to automatically scope queries.

```go
ctx = store.WithTenantID(ctx, tenantID)
// ...
tenantID := store.TenantIDFromContext(ctx)
```

---

## 5. Edition Feature Gating (Feature Toggle)

**Location:** `internal/edition/edition.go`

The `Current()` function returns either `Standard` or `Lite`. Feature gates check `edition.Current()` before enabling advanced capabilities. Lite limits are enforced at the store layer (quota checks) and tool layer (`TeamActionPolicy`).

```go
if edition.Current() == edition.Lite {
    return edition.ErrLiteLimit("max 5 agents")
}
```

---

## 6. Event Bus with Worker Pool (Pub/Sub + Worker Pool)

**Location:** `internal/eventbus/`

Typed domain events (e.g. `SessionEndedEvent`, `MemoryConsolidateEvent`) are published to a `DomainEventBus`. Workers process events in a pool with deduplication and retry. Used by memory consolidation workers.

---

## 7. Dual-DB Migration (Migration + Versioning)

**Location:** `migrations/`, `internal/store/sqlitestore/schema.go`, `internal/upgrade/version.go`

PostgreSQL migrations use `golang-migrate` sequential numbered files. SQLite uses an in-process patch map keyed by version number. Both are validated at startup against their respective schema version constants.

---

## 8. SSE Scanner (Iterator Pattern)

**Location:** `internal/providers/sse_reader.go`

A shared `SSEScanner` wraps an `io.ReadCloser` and iterates over Server-Sent Events. All streaming providers reuse this scanner instead of implementing their own SSE parsing.

---

## 9. Retry Decorator

**Location:** `internal/providers/` — `RetryDo()`

All outbound LLM provider HTTP calls are wrapped with `RetryDo(ctx, maxAttempts, fn)`. Retries respect context cancellation and apply exponential backoff for transient errors.

---

## 10. BatchQueue Generic (Aggregator Pattern)

**Location:** `internal/orchestration/`

`BatchQueue[T]` is a generic type for aggregating results from concurrent sub-agents. Workers submit results via `q.Submit(result)` and the orchestrator collects all results via `q.Collect()`.

```go
q := orchestration.NewBatchQueue[ChildResult](numWorkers)
for _, task := range tasks {
    go func(t Task) { q.Submit(process(t)) }(task)
}
results := q.Collect()
```

---

## 11. 3-Tier Memory (Hierarchical Cache)

**Location:** `internal/memory/`, `internal/consolidation/`

Working memory (L0) → Episodic (L1) → Semantic/KG (L2). Progressive loading with pgvector similarity search for L2. L0 always injected; higher tiers only loaded when context budget allows.

---

## 12. Composable Request Middleware (Decorator Chain)

**Location:** `internal/http/` middleware helpers

HTTP handlers are wrapped with composable middleware (auth, rate-limit, cache, service-tier check). Zero-alloc fast path for authenticated hot operations like chat completions.

---

## 13. Dual-Identity Agent (Value Object Duality)

**Location:** `docs/agent-identity-conventions.md`

Agents carry two identifiers: UUID (stable, opaque, for DB/FK/events) and `agent_key` (human-readable slug, for logs/paths/UI). This prevents log pollution with UUIDs while maintaining referential integrity in the database.
