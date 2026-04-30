# GoClaw — Developer Onboarding

## What is GoClaw?

GoClaw is a **multi-tenant AI agent gateway** that:
- Routes user messages through an 8-stage LLM pipeline
- Exposes a **WebSocket RPC** protocol + **REST HTTP API**
- Supports multiple LLM providers (Anthropic, OpenAI-compat, DashScope, Claude CLI)
- Integrates messaging channels (Telegram, Discord, WhatsApp, Feishu, Zalo)
- Ships as both a **server binary** (PostgreSQL) and a **desktop app** (SQLite via Wails)

---

## Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| Go | 1.26+ | Backend build |
| Node.js | 20+ | Web UI build |
| pnpm | 9+ | Web UI package manager (not npm) |
| Docker | 24+ | PostgreSQL + dev stack |
| Wails | v2 | Desktop app (optional) |

---

## First-Time Setup

```bash
# 1. Clone and enter the repo
git clone https://github.com/nextlevelbuilder/goclaw.git
cd goclaw

# 2. Start PostgreSQL
docker compose -f docker-compose.yml -f docker-compose.postgres.yml up -d postgres

# 3. Run the onboard wizard (creates config + first admin user)
go build -o goclaw . && ./goclaw onboard

# 4. Load environment and start the server
source .env.local && ./goclaw

# 5. (Optional) Start the web UI in dev mode
cd ui/web && pnpm install && pnpm dev
```

---

## Project Structure — Mental Model

```
goclaw/
├── main.go                     Entry point
├── cmd/                        Cobra CLI commands + gateway startup
├── internal/
│   ├── gateway/                WebSocket server + HTTP wiring
│   │   └── methods/            WS RPC handlers (chat, agents, sessions...)
│   ├── pipeline/               8-stage agent pipeline
│   ├── providers/              LLM provider adapters (Anthropic, OpenAI, etc.)
│   ├── http/                   HTTP API handlers (~130 handlers)
│   ├── store/                  Store interfaces + implementations
│   │   ├── pg/                 PostgreSQL (pgx/v5 + database/sql)
│   │   └── sqlitestore/        SQLite (modernc.org/sqlite)
│   ├── channels/               Channel adapters (Telegram, Discord, WhatsApp...)
│   ├── agent/                  Agent loop (think→act→observe), router
│   ├── memory/                 pgvector memory system
│   ├── consolidation/          Episodic/semantic/dreaming memory workers
│   ├── tools/                  Tool registry (filesystem, exec, web, subagent...)
│   ├── scheduler/              Lane-based concurrency (main/subagent/cron)
│   ├── vault/                  Knowledge Vault (wikilinks, hybrid search)
│   ├── i18n/                   Backend i18n (en/vi/zh)
│   └── ...
├── migrations/                 PostgreSQL migration files (57 migrations)
├── ui/
│   ├── web/                    React 19 web dashboard
│   └── desktop/                Wails v2 desktop app
└── tests/
    ├── invariants/             P0 — tenant isolation (blocking)
    ├── contracts/              P1 — API schema (requires server)
    ├── scenarios/              P2 — user journeys (requires server)
    └── integration/            Full integration tests (pgvector)
```

---

## Key Concepts

### Multi-Tenancy
Every resource (agents, sessions, memory) belongs to a `tenant_id`. The master scope (`store.IsMasterScope(ctx)`) is for cross-tenant admin operations only.

### Agent Types
- `open` — per-user context, 7 workspace files per user
- `predefined` — shared context + `USER.md` per-user override

### Agent Identity (Critical)
| Usage | Field |
|---|---|
| DB foreign keys, events | `agent.ID` (UUID) |
| Logs, file paths, UI | `agent.AgentKey` (slug) |

See `docs/agent-identity-conventions.md` for full rules.

### Desktop Edition (Lite)
- Build tag: `//go:build sqliteonly`
- Hard limits: 5 agents, 1 team, 5 members, 50 sessions
- No channels, no heartbeat, no multi-tenant
- Data at `~/.goclaw/data/`

---

## Development Workflows

### Run Integration Tests
```bash
docker run -d --name pgtest -p 5433:5432 \
  -e POSTGRES_PASSWORD=test -e POSTGRES_DB=goclaw_test \
  pgvector/pgvector:pg18

TEST_DATABASE_URL="postgres://postgres:test@localhost:5433/goclaw_test?sslmode=disable" \
  go test -v -tags integration -race ./tests/integration/
```

### Run Layered Test Suite
```bash
make test-invariants   # P0 — tenant isolation (must pass before merge)
make test-contracts    # P1 — API schemas (requires running server)
make test-scenarios    # P2 — user journeys (requires running server)
make test-critical     # P0 + P1 combined
```

### Apply a Database Migration
```bash
./goclaw migrate up
```

### Build the Desktop App
```bash
cd ui/desktop && wails dev -tags sqliteonly   # dev mode
make desktop-build VERSION=1.0.0              # production
```

---

## WebSocket Protocol

All WS communication uses frame type `req`/`res`/`event`. **First request must be `connect`**:

```json
{"method":"connect","params":{"token":"...","locale":"en"}}
```

See `websocket-protocol.md` for full frame spec and method reference.

---

## Where to Find Things

| Question | Look here |
|---|---|
| How does the pipeline work? | `internal/pipeline/pipeline.go` |
| Where are HTTP routes registered? | `cmd/gateway_http_wiring.go` |
| How does multi-tenancy work? | `internal/store/context.go`, `CONTRIBUTING.md` |
| Where is the agent identity logic? | `docs/agent-identity-conventions.md` |
| How do I add a new LLM provider? | `internal/providers/`, `internal/providerresolve/` |
| How does memory consolidation work? | `internal/consolidation/` |
| Where are the builtin tools? | `internal/tools/` |
| What are the Lite edition limits? | `internal/edition/edition.go` |
