# GoClaw — Codebase Structure

## Top-Level
```
main.go                  Entry point → cmd/
cmd/                     Cobra CLI commands, gateway startup
internal/                All business logic (unexported)
pkg/                     Exported wire types and browser utils
migrations/              PostgreSQL migration files (000001–000057+)
ui/web/                  React SPA (pnpm, Vite, Tailwind, Radix UI)
ui/desktop/              Wails v2 desktop app (React frontend + embedded gateway)
tests/                   Tests: contracts/, integration/, invariants/, scenarios/
skills/                  SKILL.md files for BM25 skill search
docs/                    Architecture and design docs
scripts/                 Install scripts (install-lite.sh, install-lite.ps1)
```

## internal/ Packages
| Package | Responsibility |
|---------|----------------|
| `agent/` | Agent loop (think→act→observe), router, resolver, input guard, suggestion engine |
| `bootstrap/` | System prompt files (SOUL.md, IDENTITY.md), seed data, per-user seed |
| `bus/` | Event bus system |
| `cache/` | Caching layer (LRU + Redis) |
| `channels/` | Channel manager: Telegram, Feishu/Lark, Zalo, Discord, WhatsApp |
| `config/` | Config loading (JSON5) + env var overlay |
| `consolidation/` | Memory consolidation workers (episodic, semantic, dreaming) |
| `crypto/` | AES-256-GCM encryption for API keys |
| `cron/` | Cron scheduling (at/every/cron expr) |
| `edition/` | Edition system (Lite, Standard) with feature gating |
| `eventbus/` | Domain event bus with worker pool, dedup, retry |
| `gateway/` | WS + HTTP server, client, method router + `methods/` RPC handlers |
| `hooks/` | Hook system for extensibility |
| `http/` | HTTP API (/v1/chat/completions, /v1/agents, /v1/skills, etc.) |
| `i18n/` | Message catalog T(locale, key, args...) + catalogs (en/vi/zh) |
| `knowledgegraph/` | KG storage and traversal |
| `mcp/` | Model Context Protocol bridge/server |
| `memory/` | Memory system (pgvector) |
| `orchestration/` | BatchQueue[T] generic, ChildResult, media conversion |
| `permissions/` | RBAC (admin/operator/viewer) |
| `pipeline/` | 8-stage agent pipeline: context→history→prompt→think→act→observe→memory→summarize |
| `providers/` | LLM provider adapters (Anthropic, OpenAI, DashScope, Claude CLI, ACP, Codex) |
| `providerresolve/` | Provider adapter + model registry with forward-compat resolver |
| `sandbox/` | Docker-based code execution sandbox |
| `scheduler/` | Lane-based concurrency (main/subagent/cron) |
| `sessions/` | Session management |
| `skills/` | SKILL.md loader + BM25 search |
| `store/` | Store interfaces + implementations |
| `store/base/` | Shared abstractions: Dialect interface, helpers (NilStr, BuildMapUpdate, BuildScopeClause) |
| `store/pg/` | PostgreSQL implementations (database/sql + pgx/v5 + sqlx) |
| `store/sqlitestore/` | SQLite implementations |
| `tasks/` | Task management |
| `tokencount/` | tiktoken BPE token counting |
| `tools/` | Tool registry, filesystem, exec, web, memory, subagent, MCP bridge, delegate |
| `tracing/` | LLM call tracing + optional OTel export (build-tag gated) |
| `tts/` | Text-to-Speech (OpenAI, ElevenLabs, Edge, MiniMax) |
| `updater/` | Desktop auto-update checker (Lite edition) |
| `upgrade/` | Database schema version tracking |
| `vault/` | Knowledge Vault with wikilinks, hybrid search, FS sync |
| `workspace/` | WorkspaceContext resolver |
| `security/` | Rate limiting, SSRF protection, shell deny patterns |

## pkg/
| Package | Contents |
|---------|----------|
| `pkg/protocol/` | Wire types (frames, methods, errors, events) |
| `pkg/browser/` | Browser automation (Rod + CDP) |

## Store Layer
- Interface-based: `store.SessionStore`, `store.AgentStore`, etc.
- Shared Dialect pattern in `store/base/`
- PG: `store/pg/*.go` — raw SQL with `$1,$2` params
- SQLite: `store/sqlitestore/` — schema.sql for fresh DBs + incremental patches in schema.go
- Nullable columns: `*string`, `*time.Time`, etc.
- Helpers: `BuildMapUpdate()`, `BuildScopeClause()`
