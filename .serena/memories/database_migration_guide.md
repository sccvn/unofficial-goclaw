# GoClaw — Database & Migration Guide

## PostgreSQL (Standard Edition)

### Migration Files
Location: `migrations/000001_*.sql` through `migrations/000057_*.sql` (and growing)
- `.up.sql` and `.down.sql` for each migration
- Current max: 000057

### Schema Version Tracking
- `internal/upgrade/version.go` — `RequiredSchemaVersion` constant
- Bump this when adding new PG migrations

### Running Migrations
```bash
./goclaw migrate up              # Via binary
make migrate                     # Via docker compose
```

### Indices Available (check before adding queries)
- JSONB GIN indexes (migration 000030)
- KG FTS and dedup indexes (migration 000031)
- Quota index (migration 000009)
- Vault path prefix index (migration 000049)
- Episodic search index (migration 000040)

## SQLite (Lite/Desktop Edition)

### Schema Files
- `internal/store/sqlitestore/schema.sql` — full schema for fresh DBs
- `internal/store/sqlitestore/schema.go` — `migrations` map + `SchemaVersion` constant

### Incremental Patches
When adding a schema change to SQLite:
1. Update `schema.sql` (the full create-table definitions)
2. Add patch to `migrations` map in `schema.go`
3. Bump `SchemaVersion` constant

### CRITICAL: Always Update Both
Missing SQLite migrations = desktop edition crashes on startup

## Query Patterns

```go
// PostgreSQL parameterized query
db.QueryContext(ctx, "SELECT * FROM agents WHERE tenant_id = $1 AND id = $2", tenantID, agentID)

// SQLite parameterized query
db.QueryContext(ctx, "SELECT * FROM agents WHERE tenant_id = ? AND id = ?", tenantID, agentID)
```

### Dialect Interface
`internal/store/base/` — `Dialect` interface handles PG vs SQLite differences
Use `BuildMapUpdate()` and `BuildScopeClause()` for dynamic WHERE/SET clauses

### Context Propagation to Store
```go
ctx = store.WithTenantID(ctx, tenantID)
ctx = store.WithUserID(ctx, userID)
ctx = store.WithAgentID(ctx, agentID)
ctx = store.WithAgentType(ctx, agentType)
ctx = store.WithLocale(ctx, locale)
```

### Tenant Isolation Rules
- Global tables (no `tenant_id`): guard with `requireMasterScope` / `store.IsMasterScope(ctx)`
- Tenant-scoped tables: guard with `requireTenantAdmin` + `WHERE tenant_id = $N`
- Examples of global tables: `builtin_tools`, disk config, package mgmt
