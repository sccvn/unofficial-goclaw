# GoClaw — Suggested Commands

## Building

```bash
# Standard build (no embedded UI)
go build -o goclaw .

# Build with embedded web UI (production)
make build-full            # builds UI first, then embeds

# Compile check — ALWAYS run both
go build ./...                     # PostgreSQL build
go build -tags sqliteonly ./...    # Desktop/SQLite build

# Desktop dev (Wails + SQLite)
make desktop-dev                   # wails dev -tags sqliteonly
make desktop-build VERSION=0.1.0   # build .app / .exe
make desktop-dmg VERSION=0.1.0     # create .dmg (macOS only)
```

## Running

```bash
./goclaw onboard           # Interactive setup wizard (first run)
source .env.local          # Load env vars from onboard
./goclaw                   # Start gateway

# Docker Compose (standard)
make up                    # Pull + start
make up-build              # Build from source + start
make down                  # Stop
make logs                  # Tail logs
make reset                 # Wipe volumes + restart

# Web UI dev server
make dev                   # cd ui/web && pnpm dev
```

## Database

```bash
./goclaw migrate up        # Run pending PG migrations
make migrate               # Via docker compose
```

## Testing

```bash
go test -race ./...                        # All unit tests + race detector

# Integration tests (needs pgvector pg18 on port 5433)
docker run -d --name pgtest -p 5433:5432 \
  -e POSTGRES_PASSWORD=test -e POSTGRES_DB=goclaw_test \
  pgvector/pgvector:pg18
TEST_DATABASE_URL="postgres://postgres:test@localhost:5433/goclaw_test?sslmode=disable" \
  go test -v -tags integration ./tests/integration/

# Layered tests
make test-invariants        # P0 — tenant isolation (MUST pass)
make test-contracts         # P1 — API schema validation (MUST pass)
make test-scenarios         # P2 — end-to-end journeys
make test-critical          # P0 + P1 combined (pre-merge gate)

# Hook-specific tests
make test-hooks-unit
make test-hooks-e2e
make test-hooks-chaos
make test-hooks-rbac
make test-hooks-tracing
make test-hooks             # All hook tests
```

## Quality / Static Analysis

```bash
go fix ./...               # Apply Go version upgrades (run before commit)
go vet ./...               # Static analysis
```

## Web UI

```bash
cd ui/web
pnpm install --frozen-lockfile
pnpm build                 # Production build
pnpm dev                   # Dev server
```

## Desktop UI

```bash
cd ui/desktop/frontend
pnpm install
pnpm build
```

## Full CI Check (before PR)

```bash
go fix ./...
go build ./...
go build -tags sqliteonly ./...
go vet ./...
go test -race ./...
cd ui/web && pnpm install --frozen-lockfile && pnpm build
```
