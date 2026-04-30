# ADR-008: Lane-Based Concurrency Scheduler

**Status:** Accepted  
**Date:** 2025-03-15  
**Relates To:** [docs/08-scheduling-cron.md](../08-scheduling-cron.md)

---

## Context

The GoClaw gateway handles concurrent agent sessions from multiple sources simultaneously:

- **Main lane**: User-initiated chat messages (latency-sensitive, user is waiting)
- **Subagent lane**: Child agents spawned by a parent during tool execution (background, depth-bounded)
- **Cron lane**: Scheduled tasks (background, periodic, lower priority)
- **Team lane**: Team member agent tasks (parallel work distribution)

Without a scheduler, all sessions compete equally for goroutines and LLM API rate limits. This creates starvation:
- A burst of cron jobs can block user-facing messages.
- A runaway sub-agent tree can exhaust all LLM call slots.
- A single busy tenant can starve other tenants.

### Forces

- Messages within the same session must be serialized — two messages from the same session must not be processed concurrently (LLM context corruption).
- Messages across different sessions should be processed concurrently (latency scaling).
- Sub-agent recursion depth must be bounded (prevent infinite agent trees).
- Sub-agent concurrency per session must be bounded (prevent cost explosion).
- Cron tasks must not starve user-initiated requests.
- Edition limits (Lite: max 2 concurrent subagents, depth 1) must be enforced at the scheduler level.

---

## Decision

### Four-Lane Architecture

```go
// internal/scheduler/scheduler.go
type Lane int

const (
    MainLane     Lane = iota // User-initiated messages
    SubagentLane             // Child agents
    CronLane                 // Scheduled tasks
    TeamLane                 // Team member tasks
)
```

Each lane has its own semaphore controlling maximum concurrent workers. The lane assignment determines both priority and resource limits.

### Per-Session Serialization

Within each lane, a per-session mutex ensures serial processing:

```go
type Scheduler struct {
    lanes    [4]*laneSemaphore
    sessions sync.Map  // sessionKey → *sync.Mutex
}

func (s *Scheduler) Enqueue(lane Lane, sessionKey string, fn func()) {
    s.lanes[lane].Acquire()
    defer s.lanes[lane].Release()
    
    mu := s.getSessionMu(sessionKey)
    mu.Lock()
    defer mu.Unlock()
    
    fn()
}
```

**Effect**: Two messages in the same session are serialized. Two messages in different sessions in the same lane run concurrently (up to lane limit).

### Concurrency Limits

| Lane | Standard | Lite |
|------|----------|------|
| Main | `main_concurrency` (config) | same |
| Subagent | `max_subagent_concurrent` (config) | 2 (edition limit) |
| Cron | `cron_concurrency` (config) | same |
| Team | derived from subagent + team config | restricted |

Edition limits are checked at scheduling time:

```go
if edition.Current().MaxSubagentConcurrent > 0 &&
   currentSubagents >= edition.Current().MaxSubagentConcurrent {
    return ErrSubagentLimitReached
}
```

### Depth Bounding

Sub-agent depth is tracked via context:

```go
ctx = scheduler.WithDepth(ctx, currentDepth+1)
// In sub-agent spawn:
depth := scheduler.DepthFromContext(ctx)
if depth >= maxDepth { return ErrMaxDepthReached }
```

Standard edition uses the config-defined `MaxSubagentDepth`. Lite caps at 1 regardless of config.

### Announce Queue (Sub-agent Results)

When multiple sub-agents complete work for the same parent session, their results must be delivered in order without flooding the parent's message stream. An **announce queue** (`internal/subagent/`) implements a producer-consumer pattern:

```
subagent A completes → enqueue result
subagent B completes → enqueue result
                           ↓
              announce worker (serialized per parent session)
                           ↓
              parent receives ordered batched results
```

---

## Consequences

### Positive

- User-facing messages are protected from starvation by lane separation.
- Per-session serialization eliminates context corruption without a global lock.
- Edition limits are enforced at one point (scheduler) — cannot be bypassed by calling the agent loop directly.
- The announce queue ensures predictable result ordering for multi-agent batches.

### Negative

- Four lanes add operational mental model overhead. Operators must understand which lane a given workload occupies to tune concurrency limits.
- Per-session mutex map grows unboundedly during long-running deployments. A reaping strategy (LRU eviction of idle sessions) is needed for very high session counts.
- Lane starvation can still occur within a lane if one session holds the lane semaphore for a very long LLM call. Addressed by timeouts, not by the scheduler.
- The `MainLane` limit effectively caps user-facing throughput. Setting it too low causes queueing; too high risks API rate limit exhaustion.

---

## Alternatives Considered

| Alternative | Rejected Because |
|-------------|-----------------|
| Single goroutine per session | Context leaks; goroutine pool exhaustion for high session counts |
| Shared goroutine pool (no lanes) | User messages starved by cron/subagent bursts |
| Priority queue (per-message priority) | Complex; priority inversion risks; harder to reason about edition limits |
| Kubernetes-level scheduling | External dependency; not viable for desktop edition; adds infrastructure complexity |
| Actor model (each session = actor) | Go's goroutine model is simpler and equally effective for this use case |
