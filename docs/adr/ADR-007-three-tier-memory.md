# ADR-007: Three-Tier Memory Architecture

**Status:** Accepted  
**Date:** 2025-04-01  
**Relates To:** [docs/26-memory-consolidation.md](../26-memory-consolidation.md), [docs/07-bootstrap-skills-memory.md](../07-bootstrap-skills-memory.md)

---

## Context

Agents must remember information across conversations to be useful. The challenge is that LLM context windows are limited (8K–200K tokens), conversation histories grow unboundedly, and different types of information have different retrieval patterns:

- **Recent facts** (last few turns) are always needed.
- **Session summaries** (what happened in past sessions) are needed selectively.
- **Conceptual knowledge** (entities, relationships, learned patterns) is needed for deep queries.

Dumping the entire history into every prompt is not scalable. Fetching everything from a vector DB on every turn adds latency. No memory at all makes agents feel amnesiac.

### Forces

- Context window must be managed: the pipeline must decide what to include in each prompt.
- Memory retrieval must be low-latency: blocking the `think` stage waiting for DB queries degrades UX.
- Memory consolidation (summarizing sessions, extracting entities) is LLM-intensive and must happen asynchronously, not in the critical path.
- Different agents have different memory needs: a customer support agent needs recent sessions; a research agent needs semantic knowledge.
- Desktop Lite edition has no pgvector — memory must degrade gracefully to FTS5 text search.

---

## Decision

### Three-Tier Model

```
Working Memory    — conversation messages in RunState (in-process, current turn)
Episodic Memory   — session summaries in DB (recent sessions, on-demand)
Semantic Memory   — entity graph + vector embeddings (long-term, searched by similarity)
```

#### Tier 1: Working Memory (L0)

- Stores the **current session's messages** in `RunState.Messages`.
- Includes recent episodic summaries (L0 injection) added automatically to the system prompt.
- Always available; no DB query in critical path.
- Bounded by `MaxContextTokens`; the `history` pipeline stage prunes oldest messages when the budget is exceeded.

#### Tier 2: Episodic Memory (L1)

- **Episodic summaries** stored in `episodic_summaries` table.
- Created asynchronously by the `episodicWorker` after each session ends (triggered by `SessionEndedEvent` on the `DomainEventBus`).
- Summaries are compressed session narratives ("The user asked about X. The agent resolved Y by doing Z.").
- Retrieved on demand when the agent uses the `recall_memory` tool or when the pipeline stage loads recent summaries for L0 injection.

#### Tier 3: Semantic Memory (L2)

- **Knowledge Graph** stored in `kg_entities` and `kg_relations` tables.
- **Vector embeddings** in the `memory` table (pgvector `halfvec`).
- Created asynchronously by the `semanticWorker` (entity extraction from summaries, CEL-expression-based deduplication via `dedupWorker`).
- Synthesized by the `dreamingWorker` which cross-references episodic memories to generate higher-order insights.
- Retrieved via hybrid search (BM25 + cosine similarity) when agent uses the `search_memory` tool.

### Async Consolidation Pipeline

Memory consolidation is event-driven and fully off the critical path:

```
Session ends
    │
    ▼ SessionEndedEvent (DomainEventBus)
    │
    ├──► episodicWorker  → create episodic_summary
    │         │
    │         ▼ EpisodicCreatedEvent
    │         ├──► semanticWorker → extract KG entities
    │         └──► dedupWorker    → merge duplicate entities
    │
    └──► (periodic) dreamingWorker → synthesize cross-session insights
```

Workers are registered at startup via `consolidation.Register(deps)`. Each worker subscribes to typed events via `eventbus.Subscribe[T]()`.

### Progressive Loading (L0/L1/L2)

The pipeline context stage applies progressive loading:

```
L0 (always): recent messages + auto-injected recent summaries
L1 (on demand): explicit recall of older sessions via tool
L2 (on demand): semantic search via tool
```

This ensures every prompt includes the most relevant recent context without blocking on vector search.

### Lite Degradation

On Desktop Lite (no pgvector):
- L2 semantic search falls back to SQLite FTS5 full-text search.
- `dreamingWorker` is disabled (KG is Lite-excluded).
- `semanticWorker` runs entity extraction but stores in SQLite without embeddings.

---

## Consequences

### Positive

- LLM critical path has no async DB dependency — L0 injection uses pre-computed summaries.
- Consolidation scales independently — episodic/semantic workers can be scaled out.
- The event-driven design decouples session lifecycle from memory persistence.
- Progressive loading prevents context bloat: only relevant memory enters the prompt.

### Negative

- Async consolidation introduces eventual consistency: a session that just ended won't have its summary available for the next session for a few seconds.
- Three storage tiers (working, episodic, semantic) add operational complexity — each has its own table(s), query patterns, and pruning strategy.
- The `dreamingWorker` requires LLM calls for synthesis — it must handle API failures gracefully without disrupting the session flow.
- L2 retrieval quality on SQLite FTS5 is significantly lower than pgvector KNN for complex semantic queries.

---

## Alternatives Considered

| Alternative | Rejected Because |
|-------------|-----------------|
| Full history in every prompt | Context overflow; cost scales with conversation length |
| Single-tier (vectors only) | No ordering guarantees; recent context gets diluted by older similar content |
| External vector DB (Pinecone, Weaviate) | Infrastructure dependency; adds latency; no offline support for Lite edition |
| In-process embedding cache | Memory footprint; embedding models are hundreds of MB; not viable for desktop binary |
| Synchronous consolidation (in critical path) | LLM summarization takes 2-10 seconds — unacceptable latency for the user |
