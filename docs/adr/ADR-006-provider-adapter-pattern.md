# ADR-006: LLM Provider Adapter Pattern with ModelRegistry

**Status:** Accepted  
**Date:** 2025-02-10  
**Relates To:** [docs/02-providers.md](../02-providers.md)

---

## Context

GoClaw supports multiple LLM providers: Anthropic (native HTTP + SSE), a family of OpenAI-compatible APIs (OpenAI, Gemini, DeepSeek, Groq, Mistral, DashScope, and others), and subprocess-bridged providers (ACP for Claude Code / Codex / Gemini CLI).

Each provider has a different wire protocol, authentication scheme, streaming format, error vocabulary, and model ID namespace. The pipeline must call whichever provider serves a given model without knowing provider-specific details.

### Forces

- The pipeline's `think` stage calls `provider.Stream(ctx, req, handler)` — it must not contain provider-specific branches.
- Operators configure providers at runtime (stored in `llm_providers` table); the set of active providers is not known at compile time.
- Model IDs from different providers can collide (`gpt-4o` from OpenAI vs a custom OpenAI-compat deployment also named `gpt-4o`).
- API keys are encrypted at rest (AES-256-GCM); the adapter must decrypt at call time.
- A provider may go offline; the pipeline needs retry logic uniformly applied.
- New providers must be addable without modifying existing code.

---

## Decision

### 1. `ProviderAdapter` Interface

```go
// internal/providers/adapter.go
type ProviderAdapter interface {
    Complete(ctx context.Context, req *Request) (*Response, error)
    Stream(ctx context.Context, req *Request, handler StreamHandler) error
    ModelInfo(modelID string) (*ModelInfo, bool)
    HealthCheck(ctx context.Context) error
}
```

Every provider implements this interface. The pipeline calls only this interface — never provider-specific types.

### 2. `ModelRegistry` — Forward-Compatible Resolution

```go
// internal/providerresolve/registry.go
type ModelRegistry struct {
    adapters map[string]ProviderAdapter  // keyed by provider ID
    models   map[string]string           // model_id → provider_id
}

func (r *ModelRegistry) Resolve(modelID string) (ProviderAdapter, error)
```

The registry maps model IDs to provider instances. Resolution order:
1. Exact match: `model_id` → provider
2. Prefix match: `claude-*` → Anthropic adapter
3. Default provider fallback (configured per tenant)

**Forward-compat resolver:** If a client sends `claude-opus-4-5` but the registry only knows `claude-3-opus`, the resolver finds the closest known model in the same family. This prevents hard failures when Anthropic or OpenAI release new model names before GoClaw updates its registry.

### 3. Shared `SSEScanner`

All streaming providers (Anthropic, OpenAI-compat) share the same SSE parser:

```go
// internal/providers/sse_reader.go
type SSEScanner struct { /* ... */ }
func (s *SSEScanner) Scan() bool
func (s *SSEScanner) Event() SSEEvent
```

This prevents each provider from implementing slightly different SSE parsing with subtly different edge-case handling.

### 4. `RetryDo` Wrapper

All providers wrap their HTTP calls in `RetryDo()`:

```go
// internal/providers/retry.go
func RetryDo(ctx context.Context, maxAttempts int, fn func() error) error
```

Retries on transient errors (5xx, rate limit 429 with backoff). The pipeline does not need to handle retries — they are transparent at the adapter level.

### 5. ACP (Subprocess) Providers

For Claude Code, Codex, and Gemini CLI — which run as local processes rather than HTTP endpoints — GoClaw uses the **ACP adapter** (`internal/providers/acp/`):

```
GoClaw Pipeline → ACP Adapter → subprocess (stdio JSON-RPC 2.0) → CLI tool → LLM
```

`ProcessPool` manages subprocess lifecycles. `ToolBridge` translates GoClaw tool definitions to the subprocess's expected format. The pipeline sees the same `ProviderAdapter` interface regardless.

---

## Consequences

### Positive

- Adding a new HTTP-based provider requires implementing `ProviderAdapter` (~100 lines) and registering it — no pipeline changes.
- The shared `SSEScanner` ensures consistent streaming behavior across all SSE providers.
- Forward-compat resolution prevents client breakage when new model names are released.
- `RetryDo` centralizes retry logic; each provider does not reinvent backoff.

### Negative

- Providers with unique capabilities (e.g., Anthropic extended thinking, OpenAI function-calling schemas) must expose them through shared request fields. The `Request` struct grows over time.
- The `ModelRegistry` is loaded at startup; adding a provider requires a gateway restart (no hot-swap).
- ACP subprocess providers add process management complexity — lifecycle events, crash recovery, and stdio buffering that HTTP adapters don't need.
- API key decryption happens on every call (not cached) — minor overhead per request, accepted for security (no plaintext keys in memory).

---

## Alternatives Considered

| Alternative | Rejected Because |
|-------------|-----------------|
| Direct provider SDKs (anthropic-go, openai-go) | Lock-in to SDK versioning; usually lag behind API changes; don't support all providers; SSE handling varies |
| GraphQL federation per provider | Massive complexity overhead; no standard LLM GraphQL schema exists |
| LangChain-style abstraction | Go bindings immature; abstraction leaks provider specifics anyway; adds dependency |
| gRPC with per-provider server | Overkill; adds infrastructure; subprocess providers still need stdio bridging |
