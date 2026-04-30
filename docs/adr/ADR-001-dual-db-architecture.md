# ADR-001: Dual-DB Architecture — PostgreSQL + SQLite

**Status:** Accepted  
**Date:** 2025-01-15  

---

## Context

GoClaw must serve two fundamentally different deployment scenarios:

1. **Server / SaaS** — Multi-tenant deployment on a Linux server. Operators need full SQL power: pgvector for semantic search, `pg_trgm` for BM25-style text search, row-level filtering on `tenant_id`, ACID transactions across tables, and enough concurrent connections to support dozens of active sessions.

2. **Desktop / Single-user** — A Wails v2 desktop app shipped as a single binary to end users who have no database installation skills. Requires zero-config setup: the database must be file-backed, embeddable in the process, and portable across macOS and Windows.

PostgreSQL is the clear winner for scenario 1. SQLite (via `modernc.org/sqlite`, a pure-Go port) is the only practical choice for scenario 2. The question is: **how to maintain both without forking business logic?**

### Forces

- All pipeline stages, HTTP handlers, and WS method handlers must be DB-agnostic — code duplication is a maintenance trap.
- SQL dialects differ: `$1/$2` (PG) vs `?` (SQLite), `RETURNING` clause availability, `pgvector` extension for KNN search, `WITH RECURSIVE` support.
- SQLite does not support concurrent writers; PostgreSQL supports many.
- Desktop edition excludes certain features (RBAC, KG, pgvector semantic search) regardless of DB capability.
- Migration systems are incompatible: `golang-migrate` for PG sequential files; SQLite needs an in-process patch map.

---

## Decision

Introduce a **Dialect abstraction** in `internal/store/base/` and define every store domain as a **Go interface** in `internal/store/stores.go`. Provide two independent implementations:

- `internal/store/pg/` — PostgreSQL using `database/sql` + `pgx/v5/stdlib`
- `internal/store/sqlitestore/` — SQLite using `modernc.org/sqlite`

The build target selects the implementation:

```
//go:build sqliteonly
```

When `sqliteonly` tag is active (desktop binary), only the SQLite package compiles. Without the tag, only PostgreSQL compiles. The gateway is initialized via dependency injection with whichever implementation is active.

### Dialect Interface

```go
// internal/store/base/dialect.go
type Dialect interface {
    Placeholder(n int) string   // "$1" for PG, "?" for SQLite
    ReturningID() string        // "RETURNING id" for PG, "" for SQLite
    JSONExtract(col, key string) string
    // ... (type-specific functions, UPSERT syntax, etc.)
}
```

### Migration Strategy

| DB | Mechanism | Location |
|----|-----------|----------|
| PostgreSQL | `golang-migrate` sequential files (`000001_init.up.sql` …) | `migrations/` |
| SQLite | In-process patch map (`map[int]string`) | `internal/store/sqlitestore/schema.go` |

Both validate a schema version at startup against constants in `internal/upgrade/version.go`. Mismatch fails fast with a clear error rather than silent corruption.

### Feature Exclusions (Lite/SQLite)

| Feature | PostgreSQL | SQLite (Lite) |
|---------|-----------|---------------|
| Semantic search (pgvector KNN) | ✓ | ✗ — FTS5 BM25 only |
| Knowledge Graph | ✓ | ✗ |
| RBAC | ✓ | ✗ |
| Full team orchestration | ✓ | Restricted |
| Multi-tenant isolation | ✓ | Single tenant |

---

## Consequences

### Positive

- All business logic (pipeline, handlers, tools) is DB-agnostic. One codebase produces two products.
- Adding a new store method requires writing it once for PG and once for SQLite — each in its natural dialect.
- Desktop binary is ~40MB with no external runtime dependencies.
- SQLite FTS5 provides acceptable text search for single-user workloads.

### Negative

- Any new store method must be implemented twice. Discipline required to keep both in sync.
- SQL features available in PG only (e.g., `GENERATED ALWAYS AS`, `LATERAL JOIN`, custom operators) cannot be used unless the feature is PG-only and excluded by build tag.
- pgvector KNN performance characteristics do not translate to SQLite FTS5 — tests across editions must account for different result ordering.
- The `//go:build sqliteonly` tag is easy to forget; CI must compile both targets on every PR.

### Risk Mitigations

- `go build ./...` (PG default) + `go build -tags sqliteonly ./...` both in CI.
- The `Dialect` interface is tested with both implementations against a shared contract suite.
- `RequiredSchemaVersion` (PG) and `SchemaVersion` (SQLite) constants fail fast if migrations are missed.

---

## Alternatives Considered

| Alternative | Rejected Because |
|-------------|-----------------|
| Single DB (PG only) | Desktop distribution requires zero-config; PG is unusable for end users |
| Single DB (SQLite only) | pgvector is core to semantic memory; FTS5 does not scale to multi-tenant |
| ORM (GORM, ent) | All ORM abstractions leak dialect differences; raw SQL gives full control and predictable explain plans |
| WASM Postgres (PGlite) | Not production-ready; wasm sandbox adds complexity with no benefit over `modernc.org/sqlite` |
