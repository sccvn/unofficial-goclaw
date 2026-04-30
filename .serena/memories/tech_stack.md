# GoClaw — Tech Stack

## Backend
- **Language**: Go 1.26
- **CLI**: `github.com/spf13/cobra`
- **WebSocket**: `github.com/gorilla/websocket`
- **DB driver**: `github.com/jackc/pgx/v5` (stdlib + sqlx), `modernc.org/sqlite` (desktop)
- **Migrations**: `github.com/golang-migrate/migrate/v4` (PG); custom schema.go map (SQLite)
- **LLM Providers**: Anthropic (native HTTP+SSE), OpenAI-compat, DashScope/Alibaba Qwen, Claude CLI (stdio+MCP), ACP (Anthropic Console Proxy), Codex
- **ORM**: None — raw SQL only. `$1/$2` params (PG), `?` (SQLite)
- **Caching**: `github.com/hashicorp/golang-lru/v2` + `github.com/redis/go-redis/v9`
- **Vector search**: pgvector extension on PostgreSQL 18
- **Browser automation**: `github.com/go-rod/rod`
- **Telegram**: `github.com/mymmrac/telego`
- **WhatsApp**: `go.mau.fi/whatsmeow`
- **TUI**: `github.com/charmbracelet/bubbletea` + `huh` + `lipgloss`
- **Desktop**: `github.com/wailsapp/wails/v2`
- **Keyring**: `github.com/zalando/go-keyring`
- **JS runtime**: `github.com/dop251/goja`
- **Token counting**: `github.com/pkoukk/tiktoken-go`
- **Config**: JSON5 via `github.com/titanous/json5`
- **OTel tracing**: `go.opentelemetry.io/otel` (build-tag gated)
- **Tailscale**: `tailscale.com` (build-tag `tsnet`)

## Web UI (`ui/web/`)
- React 19, Vite 6, TypeScript
- Tailwind CSS 4, Radix UI, Zustand, React Router 7
- Package manager: **pnpm** (never npm)
- i18n: `i18next`, locale files in `src/i18n/locales/{en,vi,zh}/`

## Desktop UI (`ui/desktop/frontend/`)
- React 19, Vite 6, TypeScript, Tailwind CSS 4, Zustand, Framer Motion
- Package manager: **pnpm**
- Wails bindings via `wails.getVersion()` etc.

## Infrastructure
- Docker Compose (multiple compose files in `compose.d/`, `compose.options/`)
- PostgreSQL 18 + pgvector as default DB
- Optional: Redis, Sandbox (Docker), Browser (rod), OTel, Tailscale, Claude CLI
