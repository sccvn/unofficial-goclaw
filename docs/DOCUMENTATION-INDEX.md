# GoClaw Documentation Index

**Complete reading guide for the GoClaw codebase.**

GoClaw is a multi-tenant AI agent gateway exposing WebSocket RPC and HTTP REST APIs. It routes LLM calls through an 8-stage pipeline, manages multi-tenant agent identity, and supports pluggable channels and tool systems.

---

## Quick Navigation

| I want to... | Go to |
|-------------|-------|
| Understand the big picture | [Architecture Overview](#architecture-and-design) |
| Start developing | [Developer Onboarding](onboarding.md) |
| Understand the agent loop | [Agent Loop](01-agent-loop.md) |
| Add a new LLM provider | [LLM Providers](02-providers.md) + [ADR-006](adr/ADR-006-provider-adapter-pattern.md) |
| Work with the database layer | [Store & Data Model](06-store-data-model.md) + [ADR-001](adr/ADR-001-dual-db-architecture.md) |
| Understand multi-tenancy | [Multi-Tenant Architecture](23-multi-tenant-architecture.md) + [ADR-004](adr/ADR-004-multi-tenant-isolation.md) |
| Review security model | [Security](09-security.md) |
| Deploy to production | [Deployment Guide](deployment.md) |
| Understand the desktop app | [Desktop Lite Edition](25-desktop-lite-edition.md) |
| Understand agent memory | [Memory Consolidation](26-memory-consolidation.md) + [Bootstrap, Skills & Memory](07-bootstrap-skills-memory.md) |
| Understand agent teams | [Agent Teams](11-agent-teams.md) |
| Understand delegation | [Orchestration & Delegation](27-orchestration-delegation.md) |
| Review API reference | [HTTP API](18-http-api.md) + [WebSocket RPC](19-websocket-rpc.md) |

---

## Architecture and Design

### Core Architecture
| Document | Description |
|----------|-------------|
| [00 - Architecture Overview](00-architecture-overview.md) | Component diagram, module map, data flow |
| [codebase-summary.md](codebase-summary.md) | High-level module map and TTS subsystem reference |
| [patterns.md](patterns.md) | Catalogue of recurring design patterns (strategy, chain, adapter, pub-sub) |
| [agent-identity-conventions.md](agent-identity-conventions.md) | UUID vs `agent_key` — when to use each, trap zones, examples |

### Architecture Decision Records

All significant architectural decisions are documented in [`docs/adr/`](adr/README.md).

| ADR | Decision |
|-----|---------|
| [ADR-001](adr/ADR-001-dual-db-architecture.md) | Why GoClaw supports both PostgreSQL and SQLite from one codebase |
| [ADR-002](adr/ADR-002-eight-stage-pipeline.md) | Why the agent pipeline uses Chain of Responsibility with 8 stages |
| [ADR-003](adr/ADR-003-agent-identity-dual-key.md) | Why agents have both a UUID and a human-readable key |
| [ADR-004](adr/ADR-004-multi-tenant-isolation.md) | How multi-tenant isolation is enforced at the SQL row level |
| [ADR-005](adr/ADR-005-edition-feature-gating.md) | How the Standard/Lite edition split works without forking code |
| [ADR-006](adr/ADR-006-provider-adapter-pattern.md) | Why all LLM providers share one `ProviderAdapter` interface |
| [ADR-007](adr/ADR-007-three-tier-memory.md) | Why memory is split into Working, Episodic, and Semantic tiers |
| [ADR-008](adr/ADR-008-lane-scheduler.md) | How the lane-based scheduler prevents starvation across workload types |

### Diagrams

PlantUML source files are in [`docs/architecture/`](architecture/README.md):

| File | Description |
|------|-------------|
| `c4-context.puml` | System context: users, channels, LLM providers |
| `c4-containers.puml` | Internal containers: gateway, pipeline, channels, DBs |
| `sequence-pipeline.puml` | 8-stage pipeline message flow |
| `erd-core.puml` | Core entity relationships |
| `class-store-layer.puml` | Store interface hierarchy (dual-DB) |
| `state-session.puml` | WebSocket session lifecycle |

---

## Core Subsystems

### Agent System
| Document | Description |
|----------|-------------|
| [01 - Agent Loop](01-agent-loop.md) | Think → act → observe loop, router, resolver |
| [02 - LLM Providers](02-providers.md) | Anthropic, OpenAI-compat, DashScope, ACP, Claude CLI |
| [03 - Tools System](03-tools-system.md) | Tool registry, policy engine, custom tools, credential scrubbing |
| [08 - Scheduling & Cron](08-scheduling-cron.md) | Lane scheduler, cron tasks, subagent concurrency |
| [12 - Extended Thinking](12-extended-thinking.md) | Anthropic extended thinking support |

### Pipeline
| Document | Description |
|----------|-------------|
| [ADR-002](adr/ADR-002-eight-stage-pipeline.md) | Pipeline design decisions |
| [01 - Agent Loop](01-agent-loop.md) | Detailed stage-by-stage walkthrough |
| [10 - Tracing & Observability](10-tracing-observability.md) | LLM call tracing, span storage, OTel export |
| [model-steering-system.md](model-steering-system.md) | Model selection, steering, and policy system |

### Memory and Knowledge
| Document | Description |
|----------|-------------|
| [07 - Bootstrap, Skills & Memory](07-bootstrap-skills-memory.md) | System prompts (SOUL/IDENTITY/AGENTS.md), skills, 3-tier memory |
| [14 - Skills Runtime](14-skills-runtime.md) | SKILL.md format, hot-reload, BM25 search |
| [15 - Core Skills System](15-core-skills-system.md) | Built-in skills hierarchy |
| [16 - Skill Publishing](16-skill-publishing.md) | Publishing and managing skills |
| [21 - Agent Evolution & Skill Management](21-agent-evolution-and-skill-management.md) | Self-evolution, metric-based adaptation |
| [24 - Knowledge Vault](24-knowledge-vault.md) | Wikilinks, hybrid search, FS sync |
| [26 - Memory Consolidation](26-memory-consolidation.md) | Episodic/semantic/dreaming workers, async consolidation pipeline |

### Multi-Agent Orchestration
| Document | Description |
|----------|-------------|
| [11 - Agent Teams](11-agent-teams.md) | Team model, lead/member roles, task board, mailbox |
| [13 - WS Team Events](13-ws-team-events.md) | WebSocket events for team operations |
| [27 - Orchestration & Delegation](27-orchestration-delegation.md) | `delegate` tool, `agent_links`, `BatchQueue[T]`, `ChildResult` |

---

## API and Protocol

| Document | Description |
|----------|-------------|
| [04 - Gateway Protocol](04-gateway-protocol.md) | WebSocket frame format, connection lifecycle, method router |
| [18 - HTTP API](18-http-api.md) | All `/v1/*` REST endpoints with request/response schemas |
| [19 - WebSocket RPC](19-websocket-rpc.md) | All WS methods: chat, agents, sessions, config, skills, cron, pairing |
| [20 - API Keys & Auth](20-api-keys-auth.md) | API key management, AES-256-GCM encryption, RBAC |
| [websocket-protocol.md](../websocket-protocol.md) | Wire-level WebSocket protocol specification |
| [api-reference.md](../api-reference.md) | Generated API reference |

---

## Channels and Integrations

| Document | Description |
|----------|-------------|
| [05 - Channels & Messaging](05-channels-messaging.md) | Telegram, Discord, WhatsApp, Feishu, Zalo, Slack adapters |
| [22 - Heartbeat System](22-heartbeat-system.md) | Agent proactive check-in scheduling |

---

## Security

| Document | Description |
|----------|-------------|
| [09 - Security](09-security.md) | 5-layer defense (transport, input, tool, output, isolation) |
| [20 - API Keys & Auth](20-api-keys-auth.md) | Key encryption and auth flow |
| [agent-hooks.md](agent-hooks.md) | Hook system security model (SSRF, circuit breaker, fail-closed) |
| [ADR-004](adr/ADR-004-multi-tenant-isolation.md) | Tenant isolation design |
| [CONTRIBUTING.md](../CONTRIBUTING.md) | Tenant-scope guard decision table (mandatory reading before admin write paths) |

---

## Database and Storage

| Document | Description |
|----------|-------------|
| [06 - Store & Data Model](06-store-data-model.md) | Schema, store interfaces, dual-DB pattern, agent access control |
| [ADR-001](adr/ADR-001-dual-db-architecture.md) | Why dual-DB, Dialect abstraction, migration strategy |
| [ADR-004](adr/ADR-004-multi-tenant-isolation.md) | Row-level `tenant_id` scoping design |

---

## Desktop and Editions

| Document | Description |
|----------|-------------|
| [25 - Desktop Lite Edition](25-desktop-lite-edition.md) | Wails v2 app, SQLite, keyring, paths, build, auto-update |
| [ADR-005](adr/ADR-005-edition-feature-gating.md) | Edition feature gating design |

---

## Operations and Deployment

| Document | Description |
|----------|-------------|
| [deployment.md](deployment.md) | Docker Compose, bare metal, env config, variants |
| [onboarding.md](onboarding.md) | First-time dev setup, project structure, common tasks |
| [packages-github.md](packages-github.md) | Docker image variants, CI/CD workflows, release tagging |

---

## Testing

| Document | Description |
|----------|-------------|
| [testing-automation.md](testing-automation.md) | Unit tests, integration tests, invariant/contract/scenario test layers |
| [testing-manual.md](testing-manual.md) | Manual test plans, exploratory testing guides |
| [review-checklist-go.md](review-checklist-go.md) | Go code review checklist |
| [review-checklist-ts.md](review-checklist-ts.md) | TypeScript/React code review checklist |

---

## Stakeholder and Team Reference

| Document | Description |
|----------|-------------|
| [summary-stakeholders.md](summary-stakeholders.md) | Non-technical overview for PMs and business stakeholders |
| [project-changelog.md](project-changelog.md) | Engineering changelog by feature area |
| [CLAUDE.md](../CLAUDE.md) | AI assistant conventions and mandatory rules for this codebase |
| [CONTRIBUTING.md](../CONTRIBUTING.md) | Contribution guide: migrations, security, tenant guards |

---

## Document Numbering Convention

Numbered documents (`NN-*.md`) follow a sequential scheme:

| Range | Category |
|-------|---------|
| 00–09 | Core architecture, agent loop, providers, tools, gateway |
| 10–19 | Channels, teams, skills, HTTP/WS API, auth |
| 20–27 | Multi-tenant, vault, desktop, memory, orchestration |

Note: Document 17 is intentionally reserved (future use).

---

## Reading Paths by Role

### New Backend Developer

1. [onboarding.md](onboarding.md)
2. [00 - Architecture Overview](00-architecture-overview.md)
3. [01 - Agent Loop](01-agent-loop.md)
4. [06 - Store & Data Model](06-store-data-model.md)
5. [ADR-001](adr/ADR-001-dual-db-architecture.md) + [ADR-003](adr/ADR-003-agent-identity-dual-key.md)
6. [CONTRIBUTING.md](../CONTRIBUTING.md)

### New Frontend Developer

1. [onboarding.md](onboarding.md)
2. [04 - Gateway Protocol](04-gateway-protocol.md)
3. [19 - WebSocket RPC](19-websocket-rpc.md)
4. [18 - HTTP API](18-http-api.md)
5. [review-checklist-ts.md](review-checklist-ts.md)

### Architect / Technical Lead

1. All ADRs in [docs/adr/](adr/README.md)
2. [00 - Architecture Overview](00-architecture-overview.md)
3. [patterns.md](patterns.md)
4. [09 - Security](09-security.md)
5. [23 - Multi-Tenant Architecture](23-multi-tenant-architecture.md)

### DevOps / Cloud Engineer

1. [deployment.md](deployment.md)
2. [packages-github.md](packages-github.md)
3. [10 - Tracing & Observability](10-tracing-observability.md)

### Product Owner / Stakeholder

1. [summary-stakeholders.md](summary-stakeholders.md)
2. [23 - Multi-Tenant Architecture](23-multi-tenant-architecture.md) (Mode 1 vs Mode 2 section)
3. [05 - Channels & Messaging](05-channels-messaging.md) (capabilities overview)
