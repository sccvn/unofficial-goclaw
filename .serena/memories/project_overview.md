# GoClaw — Project Overview

## Purpose
GoClaw is a **PostgreSQL multi-tenant AI agent gateway** with WebSocket RPC + HTTP API.
It routes user conversations to LLM providers, manages agent context/memory, runs tools,
orchestrates multi-agent teams, and integrates with messaging channels (Telegram, Discord, Slack, WhatsApp, Zalo, Feishu/Lark).

## Go Module
`github.com/nextlevelbuilder/goclaw` — Go 1.26

## Editions
- **Standard**: PostgreSQL 18 + pgvector. Full features.
- **Lite / Desktop**: SQLite via `modernc.org/sqlite`, build tag `sqliteonly`. Embedded in Wails v2 desktop app. Limited to 5 agents, 1 team, 5 members, 50 sessions.

## Binary Entry Point
`main.go` → `cmd/` (Cobra CLI)

## Key CLI Commands
```bash
./goclaw onboard          # Interactive setup wizard
./goclaw                  # Start gateway
./goclaw migrate up       # Run DB migrations
```

## Repositories / Branch Strategy
- `main` — stable protected; owner-only merge
- `dev`  — default target for all PRs
- Branches: `feat/`, `fix/`, `hotfix/`, `refactor/`, `docs/`
