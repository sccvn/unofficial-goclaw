---
description: 'Go backend development expertise for GoClaw: HTTP handlers, WebSocket RPC, agent pipeline stages, LLM provider adapters, store queries, and channel integrations. Provides Go idioms, error handling patterns, concurrency primitives, and GoClaw-specific conventions.'
---

# Go Gateway Backend — Development Prompt

## Core Go Conventions for GoClaw

### Error Handling
```go
// CORRECT
if errors.Is(err, store.ErrNotFound) { ... }

// WRONG
if err == store.ErrNotFound { ... }
```

### Conditionals
```go
// CORRECT — switch/case on same variable
switch req.Method {
case "connect": handleConnect(...)
case "chat": handleChat(...)
default: return ErrUnknownMethod
}

// WRONG — if/else if chain
if req.Method == "connect" { ... }
else if req.Method == "chat" { ... }
```

### Context Propagation
Always propagate tenant/user/agent context via the store helpers:
```go
ctx = store.WithTenantID(ctx, tenantID)
ctx = store.WithUserID(ctx, userID)
ctx = store.WithAgentID(ctx, agentID)
ctx = store.WithLocale(ctx, locale)
ctx = store.WithAgentType(ctx, agentType)
```

### SQL — Parameterized Queries Only
```go
// CORRECT (PostgreSQL)
row := db.QueryRowContext(ctx,
    "SELECT id FROM agents WHERE tenant_id = $1 AND agent_key = $2",
    tenantID, agentKey)

// CORRECT (SQLite)
row := db.QueryRowContext(ctx,
    "SELECT id FROM agents WHERE tenant_id = ? AND agent_key = ?",
    tenantID, agentKey)

// NEVER — SQL injection risk
query := fmt.Sprintf("SELECT id FROM agents WHERE agent_key = '%s'", agentKey)
```

## Pipeline Stage Pattern

Pipeline stages implement the `Stage` interface in `internal/pipeline/stage.go`. Always-on execution path means every stage runs unless explicitly short-circuited:

```go
type MyStage struct {
    deps *Deps
}

func (s *MyStage) Run(ctx context.Context, state *RunState) error {
    // Read from state, modify state, return nil or wrapped error
    if state.IsAborted() {
        return nil  // respect abort signal
    }
    // ... stage logic
    return nil
}
```

Stage order: `context → history → prompt → think → act → observe → memory → summarize`

## HTTP Handler Pattern

```go
func (h *Handler) handleAgentList(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    // 1. Auth + scope check
    if err := h.requireTenantAdmin(ctx); err != nil {
        writeError(w, err)
        return
    }
    // 2. Parse + validate request
    // 3. Call store
    agents, err := h.store.ListAgents(ctx, store.AgentFilter{...})
    if err != nil {
        writeError(w, err)
        return
    }
    // 4. Write response
    writeJSON(w, http.StatusOK, agents)
}
```

## Tenant Scope Guards

| Table type | Required guard |
|---|---|
| Global (no `tenant_id` column) | `requireMasterScope` |
| Tenant-scoped | `requireTenantAdmin` + `WHERE tenant_id = $N` |

```go
// Global table write (e.g. builtin_tools, system config)
if err := h.requireMasterScope(ctx); err != nil {
    return err
}

// Tenant-scoped write
if err := h.requireTenantAdmin(ctx); err != nil {
    return err
}
// Then always filter by tenant in SQL:
// WHERE tenant_id = $1
```

## Security Logging

```go
// All security events must use this pattern:
slog.WarnContext(ctx, "security.path_traversal",
    "requested_path", rawPath,
    "resolved_path", resolvedPath,
    "user_id", userID,
)
```

## Dual-Migration Checklist

When adding schema changes:
1. `migrations/000NNN_description.up.sql` — PostgreSQL DDL
2. `migrations/000NNN_description.down.sql` — reverse DDL
3. Bump `RequiredSchemaVersion` in `internal/upgrade/version.go`
4. Update `internal/store/sqlitestore/schema.sql` (full schema)
5. Add patch in `internal/store/sqlitestore/schema.go` `migrations` map
6. Bump `SchemaVersion` constant in `schema.go`

## Provider Adapter Pattern

New LLM providers implement `ProviderAdapter` in `internal/providerresolve/`. Streaming providers reuse `SSEScanner` from `internal/providers/sse_reader.go`. Always wrap HTTP calls with `RetryDo()`.

## Post-Implementation Checklist

```bash
go fix ./...
go build ./...
go build -tags sqliteonly ./...
go vet ./...
```
