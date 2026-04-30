# GoClaw — Deployment Guide

## Deployment Variants

| Variant | Use case | DB | Binary |
|---|---|---|---|
| Standard (Docker) | Production server | PostgreSQL 18 + pgvector | `goclaw` |
| Standard (bare metal) | Managed VPS | PostgreSQL 18 + pgvector | `goclaw` |
| Desktop Lite | Single-user desktop | SQLite | Wails app |

---

## Standard Deployment (Docker Compose)

### Minimal Stack (Backend + PostgreSQL)

```bash
# Pull and start
docker compose -f docker-compose.yml -f docker-compose.postgres.yml up -d --pull always

# Run migrations
docker compose -f docker-compose.yml -f docker-compose.postgres.yml \
  -f docker-compose.upgrade.yml run --rm upgrade
```

### With Optional Services

```bash
# Add browser automation (rod)
make up WITH_BROWSER=1

# Add OpenTelemetry tracing
make up WITH_OTEL=1

# Add code execution sandbox
make up WITH_SANDBOX=1

# Add Tailscale overlay network
make up WITH_TAILSCALE=1

# Add Redis caching
make up WITH_REDIS=1

# Add Claude CLI stdio bridge
make up WITH_CLAUDE_CLI=1

# Separate nginx for web UI on :3000 (custom SSL / reverse proxy)
make up WITH_WEB_NGINX=1
```

### Environment Configuration

Config file: `GOCLAW_CONFIG` env var (default: `./config.json`)
Secrets: `.env.local` or environment variables — **never in config.json**

```bash
# Required env vars
DATABASE_URL=postgres://user:pass@host:5432/goclaw?sslmode=disable
GOCLAW_SECRET_KEY=<32-byte random key>

# Optional
GOCLAW_PORT=8080
GOCLAW_CONFIG=/etc/goclaw/config.json
```

---

## Building from Source

```bash
# Backend only
make build

# Backend + embedded web UI (recommended for production)
cd ui/web && pnpm install && pnpm build
make build-full

# Check current version
make version
```

---

## Database Migrations

```bash
# Apply all pending migrations
./goclaw migrate up

# Rollback one migration
./goclaw migrate down 1

# Check current schema version
./goclaw migrate version
```

**Critical:** Always run migrations before starting a new version of goclaw. The Docker stack handles this automatically via the `upgrade` service.

---

## Docker Images

Published to two registries on every release:
- `ghcr.io/nextlevelbuilder/goclaw`
- `digitop/goclaw`

| Tag | Contents |
|---|---|
| `:latest`, `:vX.Y.Z` | Backend + web UI |
| `:base`, `:vX.Y.Z-base` | Backend only |
| `:full`, `:vX.Y.Z-full` | All runtimes + pre-installed skills |
| `-web:latest` | Standalone web UI (Nginx) |
| `:beta`, `:vX.Y.Z-beta.N` | Beta builds |

---

## Release Process

### Standard Release

```bash
# 1. Merge dev → main via PR
# 2. Tag from main
git checkout main && git pull
git tag v3.1.0 && git push origin v3.1.0
# GitHub Actions release.yaml triggers automatically
```

### Beta Release

```bash
git tag v3.1.0-beta.1 && git push origin v3.1.0-beta.1
# GitHub Actions release-beta.yaml triggers
```

### Desktop Release

```bash
git tag lite-v1.2.0 && git push origin lite-v1.2.0
# GitHub Actions release-desktop.yaml triggers
# Produces: macOS arm64, macOS amd64, Windows installers
```

### Tag Safety Rules

- `v3.0.0` — only triggers `release.yaml` (stable)
- `v3.0.0-beta.1` — only triggers `release-beta.yaml`
- `lite-v1.0.0` — only triggers `release-desktop.yaml`
- No pattern overlaps between the three workflows

---

## Desktop App Deployment (Lite Edition)

```bash
# Build macOS .app
make desktop-build VERSION=1.2.0

# Build macOS .dmg installer
make desktop-dmg VERSION=1.2.0
```

Desktop app data locations:
- Config / DB: `~/.goclaw/data/`
- Workspace (agent files): `~/.goclaw/workspace/`
- Secrets: OS keyring (with file fallback at `~/.goclaw/secrets/`)
- Listening port: `localhost:18790` (configurable via `GOCLAW_PORT`)

---

## Health Check

```bash
curl http://localhost:8080/health
# {"status":"ok","version":"v3.0.0"}
```

---

## Logs

```bash
# Docker
make logs

# Follow specific container
docker compose -f docker-compose.yml logs -f goclaw

# Security events specifically
docker compose logs goclaw | grep '"security.'
```
