# GoClaw — Code Style & Conventions

## Go Conventions

### General
- Raw SQL everywhere — NO ORM. `$1,$2,...` for PostgreSQL, `?` for SQLite
- Nullable columns → pointer types: `*string`, `*int`, `*time.Time`
- Error handling: always check; use `errors.Is(err, sentinel)` not `==`
- Prefer `switch/case` over `if/else if` chains on same variable
- Use `append(dst, src...)` for slice concat, not loops
- `context.Context` first argument in every function that touches DB/IO

### Naming
- Agent identity: UUID for DB/FK/events; `agent_key` for logs/paths/UI
- Dual-identity pattern applies to agents, teams, tenants
- Context keys stored in dedicated `context.go` per package (check existing pattern)

### Store Layer
- All store methods receive `context.Context` as first arg
- Context propagation via: `store.WithAgentType(ctx)`, `store.WithUserID(ctx)`, `store.WithAgentID(ctx)`, `store.WithLocale(ctx)`, `store.WithTenantID(ctx)`
- SQL injection prevention: ALWAYS use parameterized queries, never string concat
- Before adding new query: check if data is already fetched earlier in the flow
- Helpers: `BuildMapUpdate()`, `BuildScopeClause()` in `store/base/`

### Security
- All security logs: `slog.Warn("security.*")`
- AES-256-GCM encryption for stored API keys (`internal/crypto/`)
- Rate limiting, input guard (detection-only), CORS, SSRF protection, path traversal prevention
- Tenant-scope guards: `RoleAdmin` ≠ tenant check. Global tables → `requireMasterScope`. Tenant tables → `requireTenantAdmin` + `WHERE tenant_id = $N`

### Migrations (Dual-DB Rule — ALWAYS UPDATE BOTH)
- **PostgreSQL**: add SQL in `migrations/` + bump `RequiredSchemaVersion` in `internal/upgrade/version.go`
- **SQLite**: update `internal/store/sqlitestore/schema.sql` (full schema) + add incremental patch in `schema.go` `migrations` map + bump `SchemaVersion` constant

### i18n (User-Facing Strings)
- Backend: add key to `internal/i18n/keys.go` + translations to `catalog_en.go`, `catalog_vi.go`, `catalog_zh.go`
- Web UI: add to ALL locale JSON files in `ui/web/src/i18n/locales/{en,vi,zh}/`
- Bootstrap templates (SOUL.md, IDENTITY.md): English-only (LLM consumption)
- Add i18n key BEFORE writing handler code

### Build Tags
- `embedui` — embed web UI in binary
- `sqliteonly` — desktop/lite build (SQLite, no PG)
- `tui` — Bubble Tea enhanced CLI
- `tsnet` — Tailscale integration (build from source only)

## Web UI Conventions (React/TypeScript)

### Mobile-First Rules
- Viewport height: `h-dvh` NEVER `h-screen`
- Input font-size: `text-base md:text-sm` (16px on mobile → prevents iOS auto-zoom)
- Safe areas: `safe-top`, `safe-bottom`, etc. on edge-anchored elements
- Touch targets: ≥44px hit area (`@media (pointer: coarse)` with `::after`)
- Wrap `<table>` in `<div className="overflow-x-auto">` + `min-w-[600px]`
- Grid: mobile-first `grid-cols-1 sm:grid-cols-2 lg:grid-cols-N`
- Dialogs: full-screen on mobile (`max-sm:inset-0`), centered desktop (`sm:max-w-lg`)

### State & Routing
- Route params as source of truth — `useParams()`, NOT `useState` duplicate
- Optional params (`/chat/:sessionKey?`) instead of two separate routes
- `ErrorBoundary key={stableErrorBoundaryKey(pathname)}` — strips dynamic segments
- Portal dropdowns inside Radix Dialogs: add `pointer-events-auto`

### Misc
- Timezone: stored in Zustand (`useUiStore`). Charts use `formatBucketTz()` with `Intl.DateTimeFormat`
- Package manager: **pnpm** always (never npm)
