---
applyTo: '**'
---

# HIVE + Hierarchical Multi-Agent Coordination — GoClaw

## Model: HIVE

Each agent operates as an independent unit with a bounded responsibility. The `ai-augmented-engineer-v2` acts as the top-level orchestrator that delegates work to specialized agents and validates outputs before merge.

## Topology: Hierarchical

```
AI Augmented Engineer (Queen)
         │
         ├─── Lead Solution Architect
         │         │
         │         ├── Senior Backend Developer     (Go, pipeline, HTTP, WS)
         │         ├── Senior DBA                   (PG + SQLite, migrations)
         │         └── Senior Cloud Architect        (Docker, CI/CD, releases)
         │
         └─── Quality Guild
                   ├── Senior Automation Tester     (unit, integration, invariant)
                   ├── Senior Manual Tester         (test plans, mobile, channels)
                   └── Senior Frontend Developer    (React, Tailwind, Wails)
```

## Workflow Protocol

### Feature Development Flow

```
1. @lead-Solution_Architect — design the feature, write ADR if needed
2. @senior-Backend_Developer — implement Go backend changes
3. @senior-Software_Engineer-DBA — write migrations, review SQL
4. @senior-Frontend_Developer — implement UI changes
5. @senior-Automation_Tester — write tests for the feature
6. @senior-Manual_Tester — create test plan + acceptance criteria
7. @senior-Cloud_Architect — update CI/CD if needed
```

### Bug Fix Flow

```
1. @senior-Backend_Developer or @senior-Frontend_Developer — identify + fix
2. @senior-Automation_Tester — add regression test
3. @senior-Manual_Tester — verify fix manually if user-facing
```

### Release Flow

```
1. @senior-Cloud_Architect — verify workflow config + tag pattern
2. @ai-augmented-engineer-v2 — generate release notes from CHANGELOG
3. Push tag: git tag v3.X.0 && git push origin v3.X.0
```

## Delegation Rules

- An agent **must not** modify files outside its domain without explicit instruction
- Backend Developer must not modify `ui/` files
- Frontend Developer must not modify `internal/` Go files
- DBA must not modify pipeline or handler logic — only store + migration files
- All agents must consult `CLAUDE.md` and `CONTRIBUTING.md` before implementing anything that touches security, i18n, or multi-tenancy

## Quality Gates (All Agents)

Before any agent marks work complete:

- [ ] `go build ./...` passes
- [ ] `go build -tags sqliteonly ./...` passes (if Go files changed)
- [ ] `go vet ./...` passes
- [ ] `pnpm build` passes in the affected UI directory (if UI files changed)
- [ ] No new `TODO` or `FIXME` without a linked issue
- [ ] No hardcoded secrets, API keys, or passwords
- [ ] Security-sensitive changes reviewed by `@lead-Solution_Architect`
