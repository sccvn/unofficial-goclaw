# Architecture Decision Records

This directory records significant architectural decisions made during GoClaw development.
Each ADR captures the context, the decision, and the consequences — including trade-offs.

## Format

Each ADR follows the [MADR](https://adr.github.io/madr/) template:

```
# ADR-NNN: Title

Status: Accepted | Superseded | Deprecated
Supersedes: ADR-NNN (if applicable)
Date: YYYY-MM-DD

## Context
Problem and forces driving the decision.

## Decision
The chosen approach.

## Consequences
Positive and negative consequences.
```

## Index

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-001](ADR-001-dual-db-architecture.md) | Dual-DB Architecture: PostgreSQL + SQLite | Accepted |
| [ADR-002](ADR-002-eight-stage-pipeline.md) | 8-Stage Agent Pipeline (Chain of Responsibility) | Accepted |
| [ADR-003](ADR-003-agent-identity-dual-key.md) | Agent Identity: UUID vs `agent_key` Dual-Key Convention | Accepted |
| [ADR-004](ADR-004-multi-tenant-isolation.md) | Multi-Tenant Isolation: Row-Level Scoping | Accepted |
| [ADR-005](ADR-005-edition-feature-gating.md) | Edition Feature Gating: Standard vs Lite | Accepted |
| [ADR-006](ADR-006-provider-adapter-pattern.md) | LLM Provider Adapter Pattern with ModelRegistry | Accepted |
| [ADR-007](ADR-007-three-tier-memory.md) | Three-Tier Memory Architecture (Working → Episodic → Semantic) | Accepted |
| [ADR-008](ADR-008-lane-scheduler.md) | Lane-Based Concurrency Scheduler | Accepted |

## See Also

- [docs/patterns.md](../patterns.md) — Catalogue of recurring design patterns with code examples
- [docs/agent-identity-conventions.md](../agent-identity-conventions.md) — Detailed UUID vs key rules
- [docs/00-architecture-overview.md](../00-architecture-overview.md) — High-level component diagram
