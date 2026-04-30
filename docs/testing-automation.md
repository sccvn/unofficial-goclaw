# GoClaw — Testing Guide (Automation)

## Test Architecture

GoClaw uses a 4-layer test pyramid:

```
P0 — Invariants    tests/invariants/    Tenant isolation, RBAC — BLOCKING before merge
P1 — Contracts     tests/contracts/     API schema validation — requires running server
P2 — Scenarios     tests/scenarios/     User journeys — requires running server
    Integration    tests/integration/   Full DB + pipeline — requires pgvector Docker
    Unit           co-located *_test.go Fast, no I/O
```

---

## Running Tests

### Quick Commands

```bash
# All unit tests
go test ./...

# With race detector (required for CI)
go test -race ./...

# P0 invariants only (run before every merge)
make test-invariants

# P0 + P1 combined (pre-merge gate)
make test-critical

# P1 contracts (requires running server at localhost:8080)
make test-contracts

# P2 scenarios (requires running server)
make test-scenarios

# React vitest (web UI)
cd ui/web && pnpm test
cd ui/web && pnpm test:coverage
```

### Integration Tests

Require Docker with pgvector:

```bash
docker run -d --name pgtest -p 5433:5432 \
  -e POSTGRES_PASSWORD=test \
  -e POSTGRES_DB=goclaw_test \
  pgvector/pgvector:pg18

TEST_DATABASE_URL="postgres://postgres:test@localhost:5433/goclaw_test?sslmode=disable" \
  go test -v -tags integration -race ./tests/integration/
```

---

## Writing Unit Tests (Go)

### File Placement

Co-locate `_test.go` files next to the implementation:
```
internal/pipeline/think_stage.go
internal/pipeline/think_stage_test.go   ← unit test here
```

### Test Naming Convention

```go
func TestThinkStage_WhenLLMReturnsToolCall_PopulatesRunStateToolCalls(t *testing.T) {
```

Pattern: `Test<Subject>_When<Condition>_<ExpectedBehavior>`

### Mock Store Pattern

```go
// Use testutil helpers or hand-rolled mocks:
import "github.com/nextlevelbuilder/goclaw/internal/testutil"

func TestMyHandler(t *testing.T) {
    store := testutil.NewMockAgentStore()
    store.OnGetAgent = func(ctx context.Context, id uuid.UUID) (*store.Agent, error) {
        return &store.Agent{ID: id, Name: "test"}, nil
    }
    handler := NewHandler(store)
    // ...
}
```

### No Network I/O in Unit Tests

```go
// CORRECT — mock the provider
provider := &mockProvider{
    completeFunc: func(ctx context.Context, req CompletionRequest) (*CompletionResponse, error) {
        return &CompletionResponse{Content: "hello"}, nil
    },
}

// WRONG — real HTTP call in unit test
resp, err := http.Get("https://api.anthropic.com/v1/messages")
```

---

## Writing Integration Tests (Go)

Tag with `//go:build integration` — never runs in normal `go test ./...`:

```go
//go:build integration

package integration_test

import (
    "testing"
    "os"
)

func TestAgentCRUD_Integration(t *testing.T) {
    dsn := os.Getenv("TEST_DATABASE_URL")
    if dsn == "" {
        t.Skip("TEST_DATABASE_URL not set")
    }
    // ... test against real DB
}
```

---

## Writing Invariant Tests (P0)

Invariants test cross-cutting security properties. Failure blocks merge.

Example: tenant isolation invariant
```go
// tests/invariants/tenant_isolation_test.go
func TestTenantIsolation_AgentFromTenantA_NotVisibleToTenantB(t *testing.T) {
    // Create agent in Tenant A
    // Query as Tenant B
    // Assert: not found (no data leakage)
}
```

---

## Writing Contract Tests (P1)

Contract tests validate HTTP API responses against the OpenAPI spec at `internal/http/openapi_spec.json`.

```go
// tests/contracts/agents_contract_test.go
func TestAgentListEndpoint_MatchesOpenAPISpec(t *testing.T) {
    resp := httptest.NewRecorder()
    // Call handler
    // Validate response body matches schema
}
```

---

## Writing React Component Tests (vitest)

```tsx
// ui/web/src/__tests__/AgentCard.test.tsx
import { render, screen } from '@testing-library/react'
import { AgentCard } from '@/components/AgentCard'

describe('AgentCard', () => {
  it('displays agent name', () => {
    render(<AgentCard agent={{ id: '1', name: 'My Agent', agentKey: 'my-agent' }} />)
    expect(screen.getByText('My Agent')).toBeInTheDocument()
  })
})
```

Mock API adapters, not fetch:
```tsx
vi.mock('@/api/agents', () => ({
  listAgents: vi.fn().mockResolvedValue([{ id: '1', name: 'Test' }])
}))
```

---

## What NOT to Write

- No load tests (`k6`, `vegeta`, `wrk` scripts) in regular feature work
- No `runtime.ReadMemStats`-based memory leak assertions
- No p95/p99 latency assertions
- No stress tests

These flake on shared CI runners. Request explicitly if needed for a specific investigation.
