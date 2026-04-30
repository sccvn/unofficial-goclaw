# ADR-005: Edition Feature Gating — Standard vs Lite

**Status:** Accepted  
**Date:** 2025-03-01  
**Relates To:** [docs/25-desktop-lite-edition.md](../25-desktop-lite-edition.md)

---

## Context

GoClaw ships in two product configurations:

1. **Standard** — Full-featured server binary for teams and SaaS deployments. Uses PostgreSQL with pgvector, supports all channels, RBAC, Knowledge Graph, unlimited agents/teams/sessions.

2. **Lite (Desktop)** — Single-user Wails v2 desktop app. No PostgreSQL required. Ships as a native binary for macOS and Windows. Must be useful out-of-the-box for individuals without infrastructure knowledge.

The two configurations share ~95% of the codebase. The challenge is to enforce Lite restrictions without scattering `if desktop { ... }` conditionals across thousands of call sites.

### Forces

- Lite limits must be enforced at the store layer (quota checks) AND at the tool layer (blocking disallowed actions) — defense in depth.
- Certain features are architecturally incompatible with SQLite (e.g., pgvector KNN) regardless of edition.
- Adding a new edition (e.g., "Team" tier) must not require code changes outside `internal/edition/`.
- Feature gates must be readable by non-DB code paths (HTTP handlers, tool dispatchers, pipeline stages).
- The `//go:build sqliteonly` tag already gates the DB implementation — edition gating handles *feature behavior* above the DB layer.

---

## Decision

### 1. `Edition` Struct as Feature Manifest

```go
// internal/edition/edition.go
type Edition struct {
    Name                  string
    MaxAgents             int          // 0 = unlimited
    MaxTeams              int
    MaxTeamMembers        int
    MaxChannels           map[string]int
    MaxSubagentConcurrent int
    MaxSubagentDepth      int
    KGEnabled             bool
    RBACEnabled           bool
    TeamFullMode          bool         // false = lite task actions only
    VectorSearch          bool         // false = FTS5 only
}
```

Two presets are defined: `edition.Standard` (all features on, zero limits) and `edition.Lite` (limits set, advanced features off). New editions add a new preset struct — no changes needed elsewhere.

### 2. Atomic Global Singleton

```go
var current atomic.Pointer[Edition]

func Current() Edition { return *current.Load() }
func SetCurrent(e Edition) { current.Store(&e) }
```

Set once at startup in `main.go` based on the build tag:

```go
//go:build sqliteonly
func init() { edition.SetCurrent(edition.Lite) }
```

### 3. Enforcement Sites

| Layer | Mechanism | Example |
|-------|-----------|---------|
| Store (quota) | Check `edition.Current().MaxAgents` before INSERT | `AgentStore.CreateAgent()` returns `ErrLiteLimit` |
| Store (feature) | Return `ErrNotSupported` for PG-only features | `MemoryStore.SearchKNN()` → stub in SQLite |
| Tool dispatcher | `TeamActionPolicy` checks `edition.Current().TeamFullMode` | Blocks `comment`, `review`, `approve` on Lite |
| HTTP handler | `edition.Current().RBACEnabled` guard | `/v1/roles` returns 403 on Lite |
| Skills system | `edition.Current().KGEnabled` | KG-linked skill queries disabled on Lite |

### 4. `ErrLiteLimit` Sentinel

```go
func ErrLiteLimit(reason string) error {
    return fmt.Errorf("lite edition limit: %s", reason)
}
```

Callers use `errors.Is()` to detect and convert to user-facing messages via `internal/i18n`.

---

## Consequences

### Positive

- All feature limits are declared in one place (`edition.go`) — easy to audit and adjust.
- The `atomic.Pointer` is lock-free and safe for high-concurrency reads in the hot path.
- Adding a new edition preset is a two-line change (struct literal + `SetCurrent` at startup).
- Store-layer enforcement means limits are respected even if an HTTP handler forgets to check.

### Negative

- Edition is a process-global singleton — it cannot change at runtime. This means a running instance cannot be "upgraded" from Lite to Standard without a restart. Acceptable given that the two editions compile to different binaries.
- `edition.Current()` calls scattered across store methods reduce cohesion — ideally limits live in the store's domain, not in a global package. Accepted as the cost of simplicity.
- The `MaxChannels` map is a flat count per channel type — it cannot express complex rules (e.g., "max 2 Telegram bots but only 1 can be active"). Accepted as sufficient for current Lite constraints.

---

## Alternatives Considered

| Alternative | Rejected Because |
|-------------|-----------------|
| Build-tag-only feature exclusion | Cannot express runtime limits (max agents, max teams) — compile-time exclusion is all-or-nothing |
| Database-backed feature flags | Adds a DB query on every feature check; not acceptable for Lite where SQLite reads are cheap but the abstraction overhead is not worth it |
| Per-tenant edition flags | Overkill for a two-edition system; adds tenant table column and query overhead on every request |
| Separate Lite binary with stripped code | Diverges rapidly; 5% difference does not justify maintaining two separate repos/modules |
