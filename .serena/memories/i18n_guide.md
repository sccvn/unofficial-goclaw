# GoClaw — i18n Guide

## Backend i18n

### Adding a New User-Facing String (Go)
1. Add key constant to `internal/i18n/keys.go`
2. Add English translation to `internal/i18n/catalog_en.go`
3. Add Vietnamese translation to `internal/i18n/catalog_vi.go`
4. Add Chinese translation to `internal/i18n/catalog_zh.go`

**Do this BEFORE writing handler code** — missing key = runtime crash

### Usage in Go
```go
i18n.T(locale, keys.SomeKey, arg1, arg2)
```
Locale is propagated via `store.WithLocale(ctx)`:
- WebSocket: from `connect` param `locale`
- HTTP: from `Accept-Language` header

### Supported Languages
- `en` (default)
- `vi` (Vietnamese)
- `zh` (Chinese)

## Web UI i18n

### Locale Files Location
`ui/web/src/i18n/locales/{en,vi,zh}/`
Files are namespace-split (e.g., `common.json`, `agents.json`, etc.)

### Adding a New UI String
Add the key to **all 3 locale directories** — missing translation = broken UI

### Usage in React
```tsx
import { useTranslation } from 'react-i18next'
const { t } = useTranslation('namespace')
// ...
t('key.path')
```

## What NOT to Translate
- Bootstrap templates (SOUL.md, IDENTITY.md, etc.) — English only (LLM consumption)
- Internal log messages (English only)
- Technical identifiers (agent_key, tenant slugs, etc.)
