# ADR-003: Agent Identity — UUID vs `agent_key` Dual-Key Convention

**Status:** Accepted  
**Date:** 2025-03-10  
**Relates To:** [docs/agent-identity-conventions.md](../agent-identity-conventions.md)

---

## Context

Every agent (and by extension, every team and tenant) has two identifiers:

- **`uuid.UUID`** — opaque, globally unique, assigned at creation, stored as the PK in the database.
- **`agent_key`** — human-readable, URL-safe string (e.g., `@support-bot`), user-assignable, shown in UI and logs.

Early in development both identifiers were used interchangeably across the codebase. This created several production incidents:

1. **DomainEvent AgentID mismatch** — consolidation workers received an `agent_key` string when they expected a UUID to issue a `WHERE agent_id = $1` SQL query. PostgreSQL rejected the non-UUID value with `invalid input syntax for type uuid`.
2. **Filesystem path injection** — an agent key containing `../` could escape the workspace boundary when used as a path segment without sanitization.
3. **WS event fan-out** — UI clients filtered `AgentEvent` by `AgentID` field expecting the key (human-readable) but the server was publishing a UUID.

### Forces

- The store layer requires UUIDs for foreign keys, JOINs, and efficient index scans.
- Human operators need readable identifiers in logs, system prompts, and channel bot handles.
- The compiler cannot distinguish `string` (key) from `string` (UUID string) without typed wrappers.
- `uuid.UUID` is a value type — using it eliminates a whole class of parse errors at compile time.
- Some API fields must remain stable across agent renames (e.g., HTTP heartbeat params use UUID).
- System prompt templates must use `agent_key` because the LLM reads them.

---

## Decision

Establish a **dual-key convention** enforced by code review, linting, and a runtime validator:

### Rule 1 — UUID sites (DB, events, context)

Use `uuid.UUID` (or its string representation from `.String()`) when:
- The value appears in SQL `WHERE`, `JOIN`, or `INSERT`
- The value is stored in a struct field that maps to a `uuid` DB column
- The value is set in `store.WithAgentID(ctx, ...)` context key
- The value is published in `eventbus.DomainEvent.AgentID`
- The value is sent in public HTTP/WS API params for stable identity

### Rule 2 — `agent_key` sites (display, paths, LLM)

Use the string key when:
- The value appears in a system prompt template ("You are @X")
- The value is a filesystem path segment (`agents/{key}/...`)
- The value is broadcast in `AgentEvent` to the UI (displayed in the dashboard)
- The value is a log field (`"agent"` key in slog)
- The value is passed to `store.WithAgentKey(ctx, ...)` for path resolution
- The value is the router cache lookup key

### Runtime Validator

`internal/eventbus/validate_agent_id.go` emits `slog.Warn("eventbus.non_uuid_agent_id")` whenever a non-UUID string is found in `DomainEvent.AgentID`. This catches publisher drift before it silently corrupts worker queries.

### Struct Naming Convention

Structs with both identifiers must use distinct names:

```go
type SystemPromptConfig struct {
    AgentID   string    // agent_key — shown in LLM prompt, human-readable
    AgentUUID uuid.UUID // UUID — runtime identification, never shown to LLM
    // ...
}
```

---

## Consequences

### Positive

- Source of almost all UUID/key confusion bugs is now self-documenting at the call site.
- The runtime validator provides defence-in-depth for the `DomainEvent.AgentID` string trap.
- Filesystem path injection is mitigated: `sanitizeSegment()` is applied at all path-building sites.
- UI receives readable keys in events; DB receives typed UUIDs in queries.

### Negative

- Developers must consciously choose at every new API site. Code review must check this.
- `DomainEvent.AgentID` is still a `string` (not `uuid.UUID`) due to wire-format coexistence constraints. The type lift to `uuid.UUID` is a future phase gated behind rolling-deploy tests.
- Two parallel identifier flows increase mental overhead for contributors new to the codebase.

### Trap Zones (Known)

See `docs/agent-identity-conventions.md` Section 3 for the five documented trap zones and their mitigations.

---

## Alternatives Considered

| Alternative | Rejected Because |
|-------------|-----------------|
| Single identifier (UUID only) | System prompts with UUIDs are unreadable to users and LLMs; channel bot handles must be human-typed |
| Single identifier (key only) | Keys are not globally unique; DB FKs require stable, opaque IDs; renames would break all FK references |
| Typed string wrappers (`type AgentKey string`) | Adds boilerplate everywhere; does not prevent accidental `string()` conversion; deferred to future refactor |
| Automatic coercion at store boundary | Hides bugs; a resolver that tries both formats silently accepts malformed input |
