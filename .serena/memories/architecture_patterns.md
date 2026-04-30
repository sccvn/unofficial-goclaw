# GoClaw — Architecture & Key Patterns

## WebSocket Protocol
- Frame types: `req` / `res` / `event`
- First request MUST be `connect` with params: `locale`, `tenantId` etc.
- All WS method params use **camelCase** (`teamId`, `taskId`, `sessionKey`)
- RPC handlers in `internal/gateway/methods/`

## Agent Types
- `open` — per-user context (7 context files)
- `predefined` — shared context + USER.md per-user

## 8-Stage Pipeline
`context → history → prompt → think → act → observe → memory → summarize`
- Implemented in `internal/pipeline/`
- Pluggable callbacks at each stage
- Always-on execution path

## 3-Tier Memory System
- L0 (Working): conversation buffer — auto-injected
- L1 (Episodic): session summaries — progressive loading
- L2 (Semantic): knowledge graph (pgvector) — progressive loading
- Consolidation workers in `internal/consolidation/`

## Context File System
- `agent_context_files` (agent-level)
- `user_context_files` (per-user)
- Routed via `ContextFileInterceptor` in `internal/tools/`

## Provider Architecture
- All providers use `RetryDo()` for retries
- Loaded from `llm_providers` table with encrypted API keys
- `ProviderAdapter` enables pluggable implementations
- `ModelRegistry` with forward-compat resolver
- Shared `SSEScanner` in `providers/sse_reader.go`
- Provider middleware chain: cache → service tier → request guards

## Orchestration / Delegation
- `delegate_tool.go` for inter-agent task delegation
- 3 delegation modes: auto / explicit / manual
- `BatchQueue[T]` generic in `internal/orchestration/` for result aggregation
- `AgentLinks` for agent-to-agent connections

## Domain Event Bus
- `internal/eventbus/` — typed events with worker pool, dedup, retry
- Used by consolidation pipeline and memory workers

## Knowledge Vault
- Document registry + `[[wikilinks]]` + hybrid search
- FS sync, unified search query layer above existing stores
- `internal/vault/`

## Scheduler (Lane-Based)
- 3 lanes: `main` / `subagent` / `cron`
- `internal/scheduler/`

## Security Architecture
- Input guard: detection-only mode in `internal/agent/input_guard.go`
- Shell deny patterns in `internal/tools/shell_deny_groups.go`
- SSRF protection + path traversal prevention
- Rate limiting in `internal/gateway/ratelimit.go`
- AES-256-GCM encryption in `internal/crypto/`

## Telegram Formatting
`LLM output → SanitizeAssistantContent() → markdownToTelegramHTML() → chunkHTML() → sendHTML()`
- Tables rendered as ASCII in `<pre>` tags

## Config System
- JSON5 at `GOCLAW_CONFIG` env var
- Secrets in `.env.local` or env vars — NEVER in config.json
- `internal/config/`

## Self-Evolution
- 3 stages: metrics collection → suggestion analysis → guardrail-protected apply/rollback
- `internal/agent/suggestion_engine.go`, `evolution_guardrails.go`

## Feature Gating (Edition System)
- `internal/edition/edition.go` — `edition.Current()`
- Lite: no channels, heartbeat, file storage UI, skill self-manage, KG, RBAC, multi-tenant
- Tool gating: `TeamActionPolicy` blocks comment/review/approve/reject/attach/ask_user in Lite
