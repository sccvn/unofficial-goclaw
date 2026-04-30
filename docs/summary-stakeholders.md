# GoClaw — Stakeholder Summary

**For:** Product Owners, Project Managers, Business Analysts

---

## What is GoClaw?

GoClaw is a **multi-tenant AI agent gateway** — a platform that lets organizations create, deploy, and manage AI assistants (agents) that can communicate across multiple messaging channels.

Think of it as the intelligent routing layer between your users and AI models like ChatGPT, Claude, or Qwen — with full control over what each agent knows, how it behaves, and who can access it.

---

## What Problems Does It Solve?

| Problem | How GoClaw Solves It |
|---|---|
| "We want AI assistants on Telegram, Discord, and our website" | One gateway routes to all channels from one deployment |
| "Each department needs its own AI assistant" | Multi-tenant isolation — agents, users, and data are fully separated per tenant |
| "We don't want our data going to OpenAI directly" | GoClaw sits in between; you control which provider receives what data |
| "AI responses need to follow our company guidelines" | System prompts (SOUL.md, IDENTITY.md) define agent identity and rules |
| "We need a desktop tool for offline/private use" | Desktop Lite edition — single binary, no server, local SQLite database |
| "We need audit trails and usage analytics" | Built-in session tracing, LLM call logs, and usage snapshots |

---

## Key Capabilities

### Channels Supported
- Telegram
- Discord
- WhatsApp (via WhatsApp Business API)
- Feishu / Lark
- Zalo
- Direct HTTP API (OpenAI-compatible)
- WebSocket RPC (for custom integrations)

### AI Provider Support
- Anthropic (Claude)
- OpenAI and OpenAI-compatible APIs
- Alibaba DashScope (Qwen)
- Claude CLI bridge
- Codex (OpenAI)

### Memory & Knowledge
- Agents remember conversation history across sessions
- Long-term memory via semantic search (similar to how a person recalls relevant past experience)
- Knowledge Vault: agents can reference internal documents, wikis, and linked knowledge

### Deployment Modes
| Mode | Audience | Database |
|---|---|---|
| Server (Docker/bare metal) | Teams, enterprises | PostgreSQL |
| Desktop Lite | Individual users | Local SQLite |

---

## Current Status (as of April 2026)

- **57 database migrations** — indicates a mature, actively evolving schema
- **8-stage agent pipeline** — production-hardened with retry, memory consolidation, and tool execution
- **5 channel integrations** — all production-ready
- **Multi-edition system** — Standard (full-featured) + Lite (single-user desktop)
- **i18n** — English, Vietnamese, Chinese supported
- **Test coverage** — 4-layer test pyramid: unit, integration, invariant (tenant isolation), API contract

---

## Editions & Limits

| Feature | Standard Edition | Desktop Lite |
|---|---|---|
| Agents | Unlimited | 5 max |
| Teams | Unlimited | 1 max |
| Sessions | Unlimited | 50 max |
| Channels | All 5 channels | None (local only) |
| Multi-tenant | Yes | No (single user) |
| Database | PostgreSQL | SQLite (local file) |
| Deployment | Server / Docker | macOS, Windows desktop |

---

## Roadmap Indicators

Based on recent development activity:
- Active focus on **memory consolidation** (episodic + semantic, "dreaming" workers)
- Active **Knowledge Vault** development (wikilinks, hybrid search, FS sync)
- **MCP (Model Context Protocol)** bridge — connects to external tool servers
- **Self-evolution** system — metrics → suggestions → auto-adaptation
- **Desktop auto-update** — in-app update banner for Lite edition

---

## Questions for Stakeholders

1. **Which channels** do your users need? (determines integration priority)
2. **Multi-tenant or single-tenant**? (determines Standard vs Lite, hosting model)
3. **Which LLM provider**? (Anthropic, OpenAI, self-hosted?)
4. **Data residency requirements**? (on-premise Docker vs cloud vs desktop)
5. **Compliance requirements**? (audit logs, data retention, encryption at rest)
