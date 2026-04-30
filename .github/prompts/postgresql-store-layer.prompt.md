---
description: 'PostgreSQL and SQLite store layer patterns for GoClaw: dual-DB interface implementation, parameterized queries, BuildMapUpdate/BuildScopeClause helpers, nullable column patterns, index strategy, and migration authoring. Use when implementing or reviewing store code.'
---

# PostgreSQL Store Layer — Development Prompt

## Store Interface Pattern

All stores are defined as interfaces in `internal/store/stores.go`. Both PostgreSQL (`internal/store/pg/`) and SQLite (`internal/store/sqlitestore/`) implement the same interface.

```go
// 1. Define interface in internal/store/stores.go
type AgentStore interface {
    GetAgent(ctx context.Context, id uuid.UUID) (*Agent, error)
    ListAgents(ctx context.Context, filter AgentFilter) ([]*Agent, error)
    CreateAgent(ctx context.Context, input CreateAgentInput) (*Agent, error)
    UpdateAgent(ctx context.Context, id uuid.UUID, updates map[string]any) (*Agent, error)
    DeleteAgent(ctx context.Context, id uuid.UUID) error
}

// 2. Implement in internal/store/pg/agents.go (PostgreSQL)
// 3. Implement in internal/store/sqlitestore/agents.go (SQLite)
```

## BuildMapUpdate Helper

Use `base.BuildMapUpdate()` for dynamic UPDATE queries:

```go
// In internal/store/pg/agents.go
func (s *AgentStore) UpdateAgent(ctx context.Context, id uuid.UUID, updates map[string]any) (*Agent, error) {
    setClauses, args, nextIdx := base.BuildMapUpdate(updates, 1)
    args = append(args, id)
    query := fmt.Sprintf(
        "UPDATE agents SET %s, updated_at = NOW() WHERE id = $%d RETURNING *",
        setClauses, nextIdx,
    )
    // execute query...
}
```

## BuildScopeClause Helper

Use `base.BuildScopeClause()` to apply tenant/user/agent scoping automatically from context:

```go
whereClause, args := base.BuildScopeClause(ctx, base.ScopeOpts{
    TenantID: true,
    UserID:   false,
})
// Appends WHERE tenant_id = $1 automatically
```

## Nullable Column Pattern

```go
// In Go struct — use pointer types for nullable columns
type Agent struct {
    ID          uuid.UUID  `db:"id"`
    Name        string     `db:"name"`
    Description *string    `db:"description"`  // nullable
    DeletedAt   *time.Time `db:"deleted_at"`   // nullable
    Metadata    *string    `db:"metadata"`     // nullable JSON
}

// Safe nil handling
func NilStr(s string) *string {
    if s == "" {
        return nil
    }
    return &s
}
```

## Parameterized Query Examples

```go
// PostgreSQL — $1, $2, $3 placeholders
const query = `
    SELECT id, name, agent_key, tenant_id
    FROM agents
    WHERE tenant_id = $1
      AND deleted_at IS NULL
    ORDER BY created_at DESC
    LIMIT $2 OFFSET $3
`
rows, err := db.QueryContext(ctx, query, tenantID, limit, offset)

// SQLite — ? placeholders
const query = `
    SELECT id, name, agent_key, tenant_id
    FROM agents
    WHERE tenant_id = ?
      AND deleted_at IS NULL
    ORDER BY created_at DESC
    LIMIT ? OFFSET ?
`
rows, err := db.QueryContext(ctx, query, tenantID, limit, offset)
```

## Index Strategy

| Table | Indexed columns | Reason |
|---|---|---|
| `agents` | `tenant_id`, `agent_key` | All queries filter by tenant, most by key |
| `sessions` | `tenant_id`, `agent_id`, `user_id` | High-frequency lookup |
| `messages` | `session_id`, `created_at` | Pagination by session |
| `memory_items` | `agent_id`, `tenant_id` + HNSW on vector | Vector similarity search |
| `vault_documents` | `tenant_id`, `path_prefix` | Tree navigation |
| `cron_jobs` | `tenant_id`, `next_run_at` | Scheduler polling |

## JSONB GIN Index (PostgreSQL Only)

```sql
-- For tables with JSONB columns queried by key existence or containment:
CREATE INDEX CONCURRENTLY idx_agents_metadata_gin
ON agents USING GIN (metadata jsonb_path_ops);
```

## pgvector Index (PostgreSQL Only)

```sql
-- For vector similarity search (memory, KG embeddings):
CREATE INDEX CONCURRENTLY idx_memory_items_embedding_hnsw
ON memory_items USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

## Dual Migration Checklist

```
migrations/000NNN_feature.up.sql   ← PostgreSQL DDL
migrations/000NNN_feature.down.sql ← Reverse DDL
internal/upgrade/version.go        ← Bump RequiredSchemaVersion
internal/store/sqlitestore/schema.sql ← Full schema (fresh installs)
internal/store/sqlitestore/schema.go  ← Add patch entry + bump SchemaVersion
```
