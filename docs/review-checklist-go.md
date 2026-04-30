# GoClaw — Go Code Review Checklist

Use this checklist when reviewing any Go pull request in `cmd/`, `internal/`, or `pkg/`.

---

## Security

- [ ] **No SQL injection** — all user-supplied values use `$1, $2` (PG) or `?` (SQLite) placeholders; no `fmt.Sprintf` into SQL strings
- [ ] **Path traversal prevention** — file paths from user input are validated against workspace boundary using `filepath.Clean` + prefix check; never serve files outside allowed scope
- [ ] **SSRF prevention** — user-supplied URLs are validated against SSRF allowlist before outbound HTTP; `internal/security/` helpers used
- [ ] **API key encryption** — new API keys stored via `internal/crypto/` AES-256-GCM, never plaintext in DB
- [ ] **Shell deny patterns** — new `exec` tool usage checked against `gateway_lifecycle_shell_deny_groups.go`
- [ ] **Security logs** — suspicious operations log via `slog.Warn("security.*")`
- [ ] **Secrets** — no hardcoded credentials, tokens, or passwords in source

## Multi-Tenancy

- [ ] **Global table writes** guarded by `requireMasterScope` (tables without `tenant_id` column)
- [ ] **Tenant-scoped table writes** guarded by `requireTenantAdmin` + `WHERE tenant_id = $N` in all SQL
- [ ] **Cross-tenant leakage** — no query returns rows from multiple tenants without explicit master scope
- [ ] **Agent identity** — `agent.ID` (UUID) used for FK/events; `agent.AgentKey` used for logs/paths/UI — never mixed

## Database

- [ ] **Parameterized queries only** — `$1, $2` for PG, `?` for SQLite; no string-concat queries
- [ ] **Nullable columns** use `*string`, `*time.Time`, `*int64` in Go structs
- [ ] **Dual migrations** — if schema changed: both PG migration file AND SQLite schema/patch updated
- [ ] **RequiredSchemaVersion** bumped in `internal/upgrade/version.go` if PG migration added
- [ ] **SchemaVersion** bumped in `internal/store/sqlitestore/schema.go` if SQLite patch added
- [ ] **N+1 queries** — no loop that fires one DB query per item; use batch queries or joins
- [ ] **Index coverage** — new WHERE / JOIN / ORDER BY columns have appropriate indexes

## Error Handling

- [ ] **`errors.Is()`** used instead of `err == sentinel`
- [ ] **Error wrapping** — `fmt.Errorf("context: %w", err)` used to preserve error chain
- [ ] **No swallowed errors** — `_` discarding error return is flagged; justify or fix
- [ ] **Context cancellation** checked in long-running goroutines via `ctx.Err()`

## Code Style

- [ ] **`switch/case`** used instead of `if/else if` chains on same variable
- [ ] **`append(dst, src...)`** instead of loop-based slice appending
- [ ] **Early returns** over deeply nested if-blocks
- [ ] **No unnecessary allocations** in hot paths (checked by profiler if relevant)

## Context Propagation

- [ ] Store helpers used for context: `store.WithTenantID`, `store.WithUserID`, `store.WithAgentID`, `store.WithLocale`
- [ ] Context passed as first argument to all functions that call store or external services
- [ ] No `context.Background()` used inside request handlers — always derive from `r.Context()`

## i18n

- [ ] New user-facing error messages have key in `internal/i18n/keys.go`
- [ ] Key added to all 3 catalogs: `catalog_en.go`, `catalog_vi.go`, `catalog_zh.go`
- [ ] Bootstrap templates (SOUL.md, IDENTITY.md) remain English-only

## Testing

- [ ] Unit tests present for new business logic
- [ ] No network I/O in unit tests (mock the store/provider interface)
- [ ] Integration tests tagged with `//go:build integration`
- [ ] No load/stress/benchmark tests added (unless explicitly requested)
- [ ] `go test -race ./...` passes (race condition check)

## Build Hygiene

- [ ] `go build ./...` passes
- [ ] `go build -tags sqliteonly ./...` passes (if any Go files changed)
- [ ] `go vet ./...` passes with no warnings
- [ ] `go fix ./...` applied before commit

## Pipeline & Concurrency

- [ ] New goroutines use `internal/safego/` recovery wrapper to prevent silent crashes
- [ ] Channel adapters handle reconnection and don't hold global locks
- [ ] Pipeline stages respect `state.IsAborted()` check at start of `Run()`
- [ ] Event bus subscribers are idempotent (events may be delivered more than once due to retry)
