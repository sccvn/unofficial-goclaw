# GoClaw — Task Completion Checklist

## After Every Go Code Change

```bash
go fix ./...                        # Apply Go version upgrades
go build ./...                      # PG compile check
go build -tags sqliteonly ./...     # Desktop/SQLite compile check
go vet ./...                        # Static analysis
go test -race ./...                 # Unit tests with race detector
```

## After Store/DB Changes
- [ ] Update **both** PostgreSQL (`migrations/`) and SQLite (`store/sqlitestore/schema.sql` + `schema.go`)
- [ ] Bump version: `RequiredSchemaVersion` (PG) and `SchemaVersion` (SQLite)
- [ ] Verify parameterized queries — no string concatenation in SQL
- [ ] Check for N+1 queries or unnecessary full-table scans

## After Adding User-Facing Strings
- [ ] Add key to `internal/i18n/keys.go`
- [ ] Add to `catalog_en.go`, `catalog_vi.go`, `catalog_zh.go`
- [ ] For UI strings: add to `ui/web/src/i18n/locales/{en,vi,zh}/` namespace files

## After Web UI Changes
```bash
cd ui/web && pnpm build
```
- [ ] Mobile rules: `h-dvh`, 16px inputs, safe areas, touch targets
- [ ] Tables wrapped in `overflow-x-auto`
- [ ] Responsive grids mobile-first

## After Security-Sensitive Changes
- [ ] Admin writes: correct scope guard (`requireMasterScope` vs `requireTenantAdmin`)
- [ ] All DB queries use `WHERE tenant_id = $N` for tenant-scoped tables
- [ ] Security events logged as `slog.Warn("security.*")`
- [ ] API keys encrypted with AES-256-GCM

## Before PR Submission
```bash
go fix ./...
go build ./...
go build -tags sqliteonly ./...
go vet ./...
go test -race ./...
make test-invariants    # P0 — MUST pass
make test-contracts     # P1 — MUST pass
cd ui/web && pnpm install --frozen-lockfile && pnpm build
```

## After Signature / API Changes
- [ ] Grep all callers: `grep -rn 'FunctionName' .`
- [ ] Update all callers explicitly
- [ ] Check aliases and shims

## Release Tags
- Standard: `git tag v3.x.x && git push origin v3.x.x`
- Beta: `git tag v3.x.x-beta.1 && git push origin v3.x.x-beta.1`
- Desktop: `git tag lite-v1.x.x && git push origin lite-v1.x.x`
