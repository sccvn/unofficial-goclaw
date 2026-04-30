# GoClaw — LLM Providers Reference

## Supported Providers

| Provider | Implementation | Notes |
|----------|---------------|-------|
| Anthropic | `internal/providers/anthropic.go` | Native HTTP+SSE, prompt caching support |
| OpenAI-compat | `internal/providers/openai.go` | HTTP+SSE, covers GPT, Gemini via compat |
| DashScope | `internal/providers/dashscope.go` | Alibaba Qwen models |
| Claude CLI | `internal/providers/claude_cli.go` | stdio+MCP bridge |
| ACP | `internal/providers/acp_provider.go` | Anthropic Console Proxy |
| Codex | `internal/providers/codex.go` | OpenAI Codex |

## Provider Configuration
- Stored in `llm_providers` table
- API keys encrypted with AES-256-GCM (`internal/crypto/`)
- Loaded via `ProviderAdapter` from `internal/providerresolve/`

## Embedding Providers
- `embedding_openai.go` — OpenAI embeddings
- `embedding_voyage.go` — Voyage embeddings

## SSE Streaming
- Shared `SSEScanner` in `internal/providers/sse_reader.go`
- Used by Anthropic and OpenAI streaming

## Retry Strategy
- All providers use `RetryDo()` in `internal/providers/retry.go`
- Failover logic in `internal/providers/failover.go`

## Model Registry
- `internal/providers/model_registry.go`
- Forward-compat resolver for newer model names
- `internal/providers/adapter_registry.go`

## Middleware Chain (per request)
1. Cache middleware (`middleware_cache.go`)
2. Service tier middleware (`middleware_service_tier.go`)
3. Request guards (`middleware.go`)

## Key Files for Provider Work
- `internal/providers/types.go` — shared types
- `internal/providers/capabilities.go` — per-provider capabilities
- `internal/providers/schema_cleaner.go` — JSON schema normalization
- `internal/providers/schema_normalize.go` — schema normalization
- `internal/providers/error_classify.go` — error classification
