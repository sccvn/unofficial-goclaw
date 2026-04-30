# GoClaw — GitHub Copilot Global Instructions

## Project Identity

GoClaw is a **PostgreSQL multi-tenant AI agent gateway** with WebSocket RPC + HTTP REST API.
It routes LLM calls through an 8-stage pipeline, manages multi-tenant agent identity,
and supports pluggable channels (Telegram, Discord, WhatsApp, Feishu, Zalo).

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Go 1.26, Cobra CLI, gorilla/websocket, pgx/v5, golang-migrate |
| Web UI | React 19, Vite 6, TypeScript, Tailwind CSS 4, Radix UI, Zustand, React Router 7 |
| Desktop | React 19, Wails v2, SQLite (`modernc.org/sqlite`), `//go:build sqliteonly` |
| Database (standard) | PostgreSQL 18 + pgvector |
| Database (desktop) | SQLite via `modernc.org/sqlite` |
| Package manager (UI) | `pnpm` — never `npm` |

---

## Active Agents

All agents live in `.github/agents/`. Invoke them by name in Copilot Chat using `@agent-name`.

| Agent | File | Invoke When |
|---|---|---|
| AI Augmented Engineer | `ai-augmented-engineer-v2.agent.md` | Bootstrap team agents, generate docs |
| Senior Backend Developer | `senior-Backend_Developer.agent.md` | Go code, gateway, pipeline, store layer |
| Senior Frontend Developer | `senior-Frontend_Developer.agent.md` | React web UI, Tailwind, Radix UI |
| Lead Solution Architect | `lead-Solution_Architect.agent.md` | Architecture decisions, cross-cutting concerns |
| Senior Automation Tester | `senior-Automation_Tester.agent.md` | Go tests, React vitest, integration tests |
| Senior Cloud Architect | `senior-Cloud_Architect.agent.md` | Docker, CI/CD, GitHub Actions |
| Senior Manual Tester | `senior-Manual_Tester.agent.md` | Test plans, edge-case exploration |
| Senior DBA | `senior-Software_Engineer-DBA.agent.md` | Migrations, SQL review, store patterns |

---

## Mandatory Conventions (apply to ALL agents)

### Go Backend

- **Parameterized SQL only** — never string-concat user input into SQL (`$1, $2` for PG; `?` for SQLite)
- **Dual-DB migrations always** — PG: add file in `migrations/` + bump `RequiredSchemaVersion`; SQLite: update `internal/store/sqlitestore/schema.sql` + add patch in `schema.go` + bump `SchemaVersion`
- **Nullable columns** → `*string`, `*time.Time` etc.
- **`errors.Is()`** not `err == sentinel`
- **`switch/case`** not `if/else if` chains on same variable
- **Context propagation**: `store.WithAgentType`, `store.WithUserID`, `store.WithAgentID`, `store.WithLocale`, `store.WithTenantID`
- **Tenant-scope guards**: admin writes to global tables → `requireMasterScope`; tenant-scoped tables → `requireTenantAdmin` + `WHERE tenant_id = $N`
- **Security logs**: `slog.Warn("security.*")`
- **No load/stress/benchmark tests** in regular feature work — only unit + integration + chaos
- **Post-implementation checklist**: `go fix ./...`, `go build ./...`, `go build -tags sqliteonly ./...`, `go vet ./...`

### React / TypeScript

- **`h-dvh`** not `h-screen` (mobile safe viewport)
- **Input font-size**: `text-base md:text-sm` (prevent iOS auto-zoom)
- **Touch targets**: ≥ 44px on `@media (pointer: coarse)`
- **Tables**: wrap in `<div className="overflow-x-auto">` + `min-w-[600px]`
- **Dialogs**: full-screen mobile / centered desktop pattern from `ui/dialog.tsx`
- **Route params as source of truth**: `useParams()`, not duplicate `useState`
- **Portal dropdowns in dialogs**: add `pointer-events-auto` class

### i18n

- Backend new keys: add to `internal/i18n/keys.go` + all 3 catalog files (en/vi/zh) **before** writing handler code
- Web UI new strings: add to all 3 locale dirs `ui/web/src/i18n/locales/{en,vi,zh}/`
- Bootstrap templates (SOUL.md, IDENTITY.md): English-only (LLM consumption)

### Security

- AES-256-GCM for all API key storage (see `internal/crypto/`)
- SSRF protection, path traversal prevention already in place — extend, do not bypass
- Rate limiting required for all new public HTTP endpoints
- Shell deny patterns in `gateway_lifecycle_shell_deny_groups.go` — verify before adding exec tools

---

## Architecture Reference

```
WebSocket / HTTP API
       │
       ▼
   Gateway Layer (cmd/ + internal/gateway/)
       │
       ▼
   Agent Router + Scheduler (internal/agent/ + internal/scheduler/)
       │
       ▼
   8-Stage Pipeline (internal/pipeline/)
   context → history → prompt → think → act → observe → memory → summarize
       │
       ▼
   LLM Providers (internal/providers/)
   Anthropic | OpenAI-compat | DashScope | Claude CLI | ACP | Codex
       │
       ▼
   Store Layer (internal/store/)
   PostgreSQL (pg/) | SQLite (sqlitestore/) — raw SQL, no ORM
       │
       ▼
   Side Effects
   Channels (Telegram/Discord/WhatsApp/Feishu/Zalo)
   Memory (pgvector) | Knowledge Graph | Vault | Tools
```

---

## File Paths Quick Reference

| Concern | Path |
|---|---|
| CLI entry points | `cmd/` |
| HTTP API handlers | `internal/http/` |
| WebSocket methods | `internal/gateway/methods/` |
| Pipeline stages | `internal/pipeline/` |
| Store interfaces | `internal/store/stores.go` |
| PostgreSQL impl | `internal/store/pg/` |
| SQLite impl | `internal/store/sqlitestore/` |
| PG migrations | `migrations/` |
| LLM providers | `internal/providers/` |
| Channel adapters | `internal/channels/` |
| i18n backend | `internal/i18n/` |
| i18n web | `ui/web/src/i18n/locales/` |
| Web UI source | `ui/web/src/` |
| Desktop source | `ui/desktop/` |
| Test suites | `tests/{invariants,contracts,scenarios,integration}/` |
