# Architecture Diagrams

This directory contains PlantUML diagrams for the GoClaw codebase.

## Diagrams

| File | Type | Description |
|---|---|---|
| `c4-context.puml` | C4 Context | System context — users, channels, LLM providers |
| `c4-containers.puml` | C4 Container | Internal containers — gateway, pipeline, channels, DBs |
| `sequence-pipeline.puml` | Sequence | Agent pipeline message flow (8 stages) |
| `erd-core.puml` | ERD | Core entities: tenants, agents, sessions, messages, memory |
| `class-store-layer.puml` | Class | Store interface hierarchy (dual-DB abstraction) |
| `state-session.puml` | State Machine | WebSocket session lifecycle |

## Rendering

**VS Code:** Install the [PlantUML extension](https://marketplace.visualstudio.com/items?itemName=jebbs.plantuml) and press `Alt+D` to preview.

**Online:** Paste content at [https://www.plantuml.com/plantuml/uml/](https://www.plantuml.com/plantuml/uml/)

**CLI:**
```bash
# Install plantuml
brew install plantuml  # macOS

# Render all diagrams to PNG
plantuml docs/architecture/*.puml

# Render to SVG
plantuml -tsvg docs/architecture/*.puml
```

## Adding a New Diagram

1. Create `<name>.puml` in this directory
2. Use a meaningful `title` in the diagram
3. Add it to the table above
4. For C4 diagrams, use the `!include` directive to pull in C4-PlantUML macros

## Related Documentation

- [docs/adr/](../adr/README.md) — Architecture Decision Records (why decisions were made)
- [docs/DOCUMENTATION-INDEX.md](../DOCUMENTATION-INDEX.md) — Full documentation reading guide
- [docs/patterns.md](../patterns.md) — Design patterns catalogue with code examples
