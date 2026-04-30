# ADR-004: Multi-Tenant Isolation — Row-Level Scoping

**Status:** Accepted  
**Date:** 2025-02-20  
**Relates To:** [docs/23-multi-tenant-architecture.md](../23-multi-tenant-architecture.md)

---

## Context

GoClaw serves multiple tenants (organizations, departments, or individual users) from a single deployment. Each tenant must have completely isolated:

- Agents, sessions, messages, memory
- LLM providers and API keys
- MCP servers, custom tools, skills
- Teams, members, channels
- Files (workspace directory)

The system must prevent any cross-tenant data leakage even in the presence of bugs, misconfigured permissions, or unexpected query paths.

### Deployment Reality

In practice, "multi-tenant" covers a spectrum:
- A solo developer running GoClaw with a single "master" tenant.
- A company with 10 departments as separate tenants.
- A SaaS provider embedding GoClaw as the AI engine for 1,000+ end-customer tenants.

The isolation design must work for all three without per-tenant schema partitioning (which PostgreSQL supports but adds operational complexity).

### Forces

- All tables that store tenant-scoped data must have a `tenant_id uuid NOT NULL` column indexed.
- Every SELECT/UPDATE/DELETE on tenant-scoped tables must include `WHERE tenant_id = $N`.
- Admin API paths that modify **global** configuration (e.g., builtin tools, disk config) must require "master scope" — the master tenant's identity.
- An operator-level user in Tenant A must not be able to read or write data in Tenant B, even if they somehow obtain a valid session token.
- Tenant provisioning (creating a new tenant) is a master-scope-only operation.

---

## Decision

### 1. Row-Level `tenant_id` on All Tenant-Scoped Tables

Every table that holds tenant-scoped data has:

```sql
tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
```

An index on `tenant_id` is created in every migration that adds such a column.

**Tenant-scoped tables include:** `agents`, `sessions`, `messages`, `memory`, `agent_context_files`, `user_context_files`, `llm_providers`, `mcp_servers`, `custom_tools`, `skills`, `teams`, `team_members`, `agent_links`, `kg_entities`, `kg_relations`, `episodic_summaries`, `channel_instances`, and more.

**Global tables (no `tenant_id`):** `tenants`, `builtin_tools`, `hook_definitions` (master scope only).

### 2. Context Propagation

`tenant_id` is extracted from the authenticated session token and injected into `context.Context` at the gateway boundary:

```go
ctx = store.WithTenantID(ctx, tenantID)
```

Every store method extracts it:

```go
tenantID := store.TenantIDFromContext(ctx)
// used in: WHERE tenant_id = $1
```

This makes it impossible for a store method to query data without a `tenant_id` scope — the query will fail or return empty results if the context key is absent.

### 3. Scope Guards at API Layer

Two guard functions wrap all admin write operations:

```go
// Writes to global tables (no tenant_id) → require master tenant
requireMasterScope(ctx) error

// Writes to tenant-scoped tables → require tenant admin role
requireTenantAdmin(ctx) error
```

The shared predicate:

```go
func IsMasterScope(ctx context.Context) bool {
    return store.TenantIDFromContext(ctx) == MasterTenantID
}
```

### 4. Workspace File Isolation

Each tenant's agent files live in a dedicated subdirectory:

```
workspace/
├── tenants/{tenant_id}/
│   └── agents/{agent_key}/
│       ├── SOUL.md
│       ├── IDENTITY.md
│       └── ...
```

`resolvePath()` applies `filepath.Clean()` + `HasPrefix()` to guarantee no path escapes the tenant's root.

### 5. Cascade Delete

`ON DELETE CASCADE` on all `tenant_id` foreign keys ensures that deleting a tenant removes all its data atomically, with no orphaned rows.

---

## Consequences

### Positive

- Isolation is enforced at the SQL level — business logic bugs cannot leak cross-tenant data.
- Context propagation makes `tenant_id` implicit in every store call; developers cannot accidentally omit it.
- Cascade delete simplifies tenant offboarding.
- Single schema supports any number of tenants without per-tenant schema maintenance.

### Negative

- `WHERE tenant_id = $N` adds one predicate to every query. For large deployments, this is negligible with proper indexing; without indexes it causes full table scans.
- All tenant-scoped tables must have `tenant_id` — easy to forget in new migrations. The post-implementation checklist enforces review.
- Master-scope operations (global config) require special guard calls. Forgetting `requireMasterScope` is a security bug — must be caught in code review.
- Row-level security (PostgreSQL RLS) is not used — isolation is enforced by the application layer. RLS would provide a deeper safety net but adds complexity and limits ORM/raw-SQL flexibility.

### Risk Mitigations

- `tests/invariants/` P0 test suite validates tenant isolation: creates two tenants, performs operations in Tenant A, asserts Tenant B sees nothing.
- `CONTRIBUTING.md` "Tenant-scope guards" decision table is mandatory reading before touching admin write paths.
- Linting: all new store queries reviewed for missing `tenant_id` clauses in code review.

---

## Alternatives Considered

| Alternative | Rejected Because |
|-------------|-----------------|
| PostgreSQL schemas per tenant | Operational complexity; hundreds of schemas; `golang-migrate` does not scale to dynamic schemas |
| Separate database per tenant | Extreme operational overhead; connection pool per-tenant kills resource efficiency |
| PostgreSQL Row-Level Security (RLS) | Adds complexity; requires policy per table; less transparent in raw SQL queries; deferred to future hardening |
| Application-only isolation (no DB column) | Single compromised store method could expose all tenants; not acceptable |
