# GoClaw — Desktop Edition (Lite) Reference

## Build System
- Build tag: `//go:build sqliteonly`
- Entry point: `ui/desktop/main.go` + `ui/desktop/app.go`
- Wails v2 framework: embeds gateway + React frontend in single binary

## Data Locations
- SQLite DB, configs: `~/.goclaw/data/`
- Secrets (keyring with file fallback): `~/.goclaw/secrets/`
- Agent files, team workspace: `~/.goclaw/workspace/`

## Port
- Default: `18790` (localhost only)
- Configurable via `GOCLAW_PORT`

## Version
- Set via `-ldflags` at build time
- Frontend reads via `wails.getVersion()`

## Edition Limits
- 5 agents max
- 1 team max
- 5 members max
- 50 sessions max
- No: channels, heartbeat, file storage UI, skill self-manage, KG, RBAC, multi-tenant

## Tool Gating
`TeamActionPolicy` in `internal/tools/team_action_policy.go`:
- Lite blocks: comment/review/approve/reject/attach/ask_user
- `skill_manage`/`publish_skill` NOT registered in Lite

## File Serving
2-layer path isolation in `internal/http/files.go`:
- Layer 1: workspace boundary (all editions)
- Layer 2: tenant scope (standard only, with RBAC)

## Auto-Update
- `internal/updater/updater.go`
- Checks GitHub Releases for `lite-v*` tags
- Frontend `UpdateBanner` shows notification

## Dev & Build Commands
```bash
cd ui/desktop && wails dev -tags sqliteonly     # Hot reload dev
make desktop-dev                                 # Same via Makefile
make desktop-build VERSION=0.1.0                # Build .app/.exe
make desktop-dmg VERSION=0.1.0                  # .dmg installer (macOS)
```

## Release Tag Pattern
`lite-v1.x.x` → triggers `release-desktop.yaml` CI workflow
Builds: macOS (arm64+amd64) + Windows → GitHub Release
