# 25 - Desktop Lite Edition

The Desktop Lite edition packages GoClaw as a native desktop application using **Wails v2** — a Go framework that embeds a React frontend in a native OS window without a separate browser process. The result is a single installable binary for macOS and Windows that runs the full GoClaw gateway locally with an embedded SQLite database.

---

## 1. Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│  Desktop Binary (single executable, ~40MB)                    │
│                                                               │
│  ┌─────────────────────┐    ┌─────────────────────────────┐  │
│  │  React Frontend      │    │  GoClaw Gateway (Go)        │  │
│  │  (embedded via       │◄──►│  Wails RPC + HTTP           │  │
│  │   wails.Run())       │    │  localhost:18790             │  │
│  └─────────────────────┘    └──────────┬──────────────────┘  │
│                                         │                      │
│                               ┌─────────▼─────────┐          │
│                               │  SQLite Database   │          │
│                               │  ~/.goclaw/data/   │          │
│                               └───────────────────┘          │
└──────────────────────────────────────────────────────────────┘
```

The desktop entry point is `ui/desktop/main.go`. The React frontend lives in `ui/desktop/frontend/`. The frontend is separate from the web dashboard at `ui/web/` — different structure, i18n namespaces, and component set.

### Build Tag

All desktop-specific code is guarded with:

```go
//go:build sqliteonly
```

This tag activates:
- SQLite store implementations (`internal/store/sqlitestore/`)
- `edition.Lite` preset at startup
- OS keyring for secret storage
- Desktop-specific Wails bindings

---

## 2. Edition Limits (Lite vs Standard)

| Feature | Standard | Lite |
|---------|----------|------|
| Agents | Unlimited | 5 |
| Teams | Unlimited | 1 |
| Team members | Unlimited | 5 |
| Sessions | Unlimited | 50 (soft purge) |
| Subagent concurrency | Config | 2 |
| Subagent depth | Config | 1 |
| Knowledge Graph | ✓ | ✗ |
| RBAC | ✓ | ✗ |
| Semantic search (pgvector) | ✓ | FTS5 only |
| Multi-tenant | ✓ | Single tenant |
| Channel integrations | Unlimited | 1 Telegram + 1 Discord |
| Heartbeat | ✓ | ✗ |
| Skill self-manage | ✓ | ✗ |
| File storage UI | ✓ | ✗ |
| Team full actions (comment, review, approve) | ✓ | ✗ |

Tool-level blocking is enforced by `TeamActionPolicy` in `internal/tools/team_action_policy.go`.

---

## 3. Data Storage

### Paths

| Resource | Path |
|----------|------|
| SQLite database | `~/.goclaw/data/goclaw.db` |
| Config file | `~/.goclaw/data/config.json` |
| Agent workspace files | `~/.goclaw/workspace/` |
| Secret store (fallback) | `~/.goclaw/secrets/` |

### SQLite Schema

The SQLite schema (`internal/store/sqlitestore/schema.sql`) is a translation of the PostgreSQL migrations with:

| PG Type | SQLite Equivalent |
|---------|------------------|
| `UUID` | `TEXT` (36-char RFC4122 string) |
| `TIMESTAMPTZ` | `TEXT` (RFC3339) |
| `JSONB` | `TEXT` (JSON string) |
| `BYTEA` | `BLOB` |
| `tsvector` | Omitted — FTS5 virtual tables |
| `vector(N)` | Omitted — no pgvector; basic embedding storage as BLOB |
| `text[]` / `uuid[]` | `TEXT` (JSON array) |

UUIDs are generated in Go (not database-generated). FTS5 virtual tables replace PG's `tsvector` GIN indexes for full-text search.

### Migration Strategy

SQLite uses an in-process patch map rather than `golang-migrate`:

```go
// internal/store/sqlitestore/schema.go
var migrations = map[int]string{
    1: `CREATE TABLE IF NOT EXISTS ...`,
    2: `ALTER TABLE agents ADD COLUMN ...`,
    // ...
}

const SchemaVersion = 29
```

At startup, the store applies all patches with version > current DB version, then updates the `schema_version` table. A mismatch (DB version > code version) fails fast with a clear error.

---

## 4. Secret Management

Desktop edition uses the OS keyring for secret storage via `go-keyring`, with a file fallback for environments without a keyring daemon (common on headless Linux):

```
1st: OS keyring (macOS Keychain, Windows Credential Manager, Linux libsecret)
2nd: Encrypted file at ~/.goclaw/secrets/{key}.enc (AES-256-GCM)
```

The `GOCLAW_SECRET_KEY` environment variable is stored in the keyring at first startup (onboard wizard) and retrieved on subsequent launches. No plaintext secrets are written to disk.

---

## 5. Port and Networking

| Setting | Value |
|---------|-------|
| Default port | `18790` |
| Bind address | `127.0.0.1` (localhost only) |
| Override | `GOCLAW_PORT` env var |
| External access | Not supported in Lite by design |

The gateway listens on `localhost:18790`. The Wails window communicates via Wails RPC bindings (not HTTP from the React layer) for native desktop features (app version, update checks, window controls). Chat and agent API calls use the local HTTP gateway at `http://localhost:18790/v1/`.

---

## 6. Auto-Update Checker

`internal/updater/updater.go` polls the GitHub Releases API for tags matching `lite-v*`:

```
GET https://api.github.com/repos/nextlevelbuilder/goclaw/releases
→ filter tags matching: lite-v*
→ compare with current version (set via -ldflags at build time)
```

The frontend `UpdateBanner` component shows a dismissible notification when a newer version is available. The updater does **not** auto-download or auto-install — it links to the GitHub release page.

Check frequency: once per app launch, results cached in memory for the session.

---

## 7. Wails Bindings

The desktop frontend calls native Go functions via Wails bindings (not HTTP):

```go
// ui/desktop/app.go
type App struct { /* ... */ }

func (a *App) GetVersion() string      { return cmd.Version }
func (a *App) CheckUpdate() UpdateInfo { return updater.Check() }
func (a *App) OpenURL(url string)      { /* platform open */ }
```

In the React frontend:

```typescript
import { GetVersion, CheckUpdate } from "../wailsjs/go/main/App";

const version = await GetVersion();
```

**Important**: All WS method params use **camelCase** (`teamId`, `taskId`, `sessionKey`) — match Go struct `json:"..."` tags exactly.

---

## 8. Build and Distribution

### Development

```bash
# Option 1: Wails dev server with hot reload
cd ui/desktop && wails dev -tags sqliteonly

# Option 2: Makefile shortcut
make desktop-dev
```

### Production Build

```bash
# Build .app (macOS) or .exe (Windows)
make desktop-build VERSION=1.2.0

# Create .dmg installer (macOS only)
make desktop-dmg VERSION=1.2.0
```

Version is injected via ldflags:

```
-ldflags "-X github.com/nextlevelbuilder/goclaw/cmd.Version=1.2.0"
```

### CI/CD (GitHub Actions)

Tag `lite-v*` triggers `.github/workflows/release-desktop.yaml`:

1. Build macOS arm64 (`.app` + `.dmg`)
2. Build macOS amd64 (`.app` + `.dmg`)
3. Build Windows amd64 (`.exe` installer)
4. Create GitHub Release (prerelease if tag contains `-beta` or `-rc`)

Install scripts:
- macOS: `scripts/install-lite.sh`
- Windows: `scripts/install-lite.ps1` (PowerShell)

---

## 9. Desktop vs Web Frontend Differences

| Concern | `ui/web/` | `ui/desktop/frontend/` |
|---------|-----------|----------------------|
| Framework | React 19 + Vite + Tailwind + Radix UI | React 19 + Vite + Tailwind + Framer Motion |
| Routing | React Router 7 | React Router 7 |
| State | Zustand | Zustand |
| i18n namespaces | `ui/web/src/i18n/locales/` | `ui/desktop/frontend/src/i18n/locales/` |
| Test framework | Vitest + RTL | — |
| Native calls | HTTP only | Wails bindings (`wailsjs/go/`) |
| Update banner | Not present | `UpdateBanner` component |
| Deployment | Nginx / embedded in binary | Embedded in Wails binary |

Both frontends share the same WebSocket RPC protocol and HTTP API — the backend is identical.

---

## 10. Compile-Time Validation

All CI runs must compile both targets:

```bash
# Standard (PostgreSQL) build
go build ./...

# Desktop (SQLite) build
go build -tags sqliteonly ./...

# Both must pass before merge
```

If `sqliteonly` build fails, the desktop release will be broken. The post-implementation checklist enforces this.
