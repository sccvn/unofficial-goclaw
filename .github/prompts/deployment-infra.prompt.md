---
description: 'Deployment, Docker, and CI/CD for GoClaw: multi-stage Dockerfile, docker-compose profiles, GitHub Actions workflows (ci/release/release-beta/release-desktop), release tag patterns, Docker registry (GHCR + Docker Hub), and Wails desktop builds. Use for infrastructure and release automation.'
---

# Deployment & Infrastructure — Prompt

## Docker Build Variants

| Variant | Tag | Build Command | Contents |
|---|---|---|---|
| `latest` | `:latest`, `:vX.Y.Z` | `make build-full` | Backend + web UI |
| `base` | `:base`, `:vX.Y.Z-base` | `make build` | Backend only |
| `full` | `:full`, `:vX.Y.Z-full` | Custom | All runtimes + skills |
| `web` | `-web:latest` | Web only | Standalone nginx |
| `beta` | `:beta`, `:vX.Y.Z-beta.N` | CI | Beta builds from dev |

## Dockerfile Key Patterns

```dockerfile
# Backend build — CGO_ENABLED=0 required for static binary
RUN CGO_ENABLED=0 go build -tags embedui -ldflags="-s -w -X .../cmd.Version=${VERSION}" -o goclaw .

# Web UI — pnpm, not npm
RUN corepack enable && pnpm install --frozen-lockfile
RUN pnpm build
```

## Docker Compose Profiles

```bash
# Minimal: backend + postgres
docker compose -f docker-compose.yml -f docker-compose.postgres.yml up -d

# With options (compose flags from Makefile):
make up WITH_BROWSER=1      # + rod browser
make up WITH_OTEL=1         # + OpenTelemetry collector
make up WITH_SANDBOX=1      # + Docker code sandbox
make up WITH_TAILSCALE=1    # + Tailscale overlay
make up WITH_REDIS=1        # + Redis cache
make up WITH_CLAUDE_CLI=1   # + Claude CLI stdio bridge
make up WITH_WEB_NGINX=1    # + separate nginx on :3000
```

## Release Tag Safety Matrix

| Tag Pattern | Triggers | Result |
|---|---|---|
| `v3.0.0` (clean semver) | `release.yaml` only | Stable release, Docker latest+versioned |
| `v3.0.0-beta.1` | `release-beta.yaml` only | Beta release, Docker beta tag |
| `v3.0.0-rc.1` | `release-beta.yaml` only | RC release, GitHub prerelease |
| `lite-v1.0.0` | `release-desktop.yaml` only | Desktop stable, macOS+Windows |
| `lite-v1.0.0-beta.1` | `release-desktop.yaml` only | Desktop prerelease |

**Rule: No two workflows share a tag pattern. Never create a tag that matches multiple patterns.**

## Creating Releases

```bash
# Standard release (after merging dev → main)
git tag v3.0.0 && git push origin v3.0.0

# Beta release (from dev branch)
git tag v3.0.0-beta.1 && git push origin v3.0.0-beta.1

# Desktop release
git tag lite-v1.2.0 && git push origin lite-v1.2.0

# Desktop beta
git tag lite-v1.2.0-beta.1 && git push origin lite-v1.2.0-beta.1
```

## CI Workflow (`ci.yaml`)

Triggers: push to `main`, PR to `main` or `dev`

```yaml
steps:
  - go build ./...
  - go build -tags sqliteonly ./...
  - go vet ./...
  - go test ./...
  - cd ui/web && pnpm install && pnpm build
```

## Desktop Build (Wails v2)

```bash
# Dev mode with hot reload
cd ui/desktop && wails dev -tags sqliteonly
# or via Makefile:
make desktop-dev

# Production build
make desktop-build VERSION=1.2.0

# macOS DMG installer
make desktop-dmg VERSION=1.2.0
```

Desktop app ports:
- Default: `localhost:18790`
- Configurable via `GOCLAW_PORT` env var

## Docker Registry Push

Both registries are pushed in release workflows:
- `ghcr.io/nextlevelbuilder/goclaw`
- `digitop/goclaw`

Ensure both `GHCR_TOKEN` and `DOCKERHUB_TOKEN` secrets are set in GitHub repository settings.

## Migrations on Deploy

The upgrade container runs automatically on `make up`:
```yaml
# docker-compose.upgrade.yml
services:
  upgrade:
    command: ./goclaw migrate up
```

Never deploy without running migrations. The `make up` target handles this automatically.
