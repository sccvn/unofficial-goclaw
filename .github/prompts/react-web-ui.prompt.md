---
description: 'React 19 + TypeScript web UI development for GoClaw dashboard (ui/web/) and desktop frontend (ui/desktop/frontend/). Provides mobile UX rules, component patterns, Zustand state, routing conventions, i18n, and Radix UI usage specific to this codebase.'
---

# React Web UI — Development Prompt

## Mobile UX Rules (Non-Negotiable)

```tsx
// CORRECT viewport height
<div className="h-dvh flex flex-col">

// WRONG — breaks on mobile browsers
<div className="h-screen flex flex-col">
```

```tsx
// CORRECT — prevent iOS Safari zoom
<input className="text-base md:text-sm" />
<textarea className="text-base md:text-sm" />

// WRONG — triggers iOS auto-zoom
<input className="text-sm" />
```

```tsx
// Tables MUST be scrollable on mobile
<div className="overflow-x-auto">
  <table className="min-w-[600px]">
    ...
  </table>
</div>
```

## Routing — useParams as Source of Truth

```tsx
// CORRECT — derive from URL, not duplicate state
function ChatPage() {
  const { sessionKey } = useParams<{ sessionKey: string }>();
  // Use sessionKey directly — don't copy to useState
}

// WRONG — causes race conditions + UI flash
function ChatPage() {
  const { sessionKey: paramKey } = useParams();
  const [sessionKey, setSessionKey] = useState(paramKey); // DON'T DO THIS
}
```

## ErrorBoundary Key — Stable, Not Pathname

```tsx
// CORRECT — strip dynamic segments for stable key
<ErrorBoundary key={stableErrorBoundaryKey(pathname)}>
  <Outlet />
</ErrorBoundary>

// WRONG — causes full remount on every param change
<ErrorBoundary key={location.pathname}>
  <Outlet />
</ErrorBoundary>
```

## Dialog Pattern

```tsx
// Full-screen on mobile, centered on desktop (matches ui/dialog.tsx)
<Dialog>
  <DialogContent className="max-sm:inset-0 sm:max-w-lg">
    ...
  </DialogContent>
</Dialog>
```

## Portal Dropdown Inside Dialog

```tsx
// When using createPortal for dropdown inside Radix Dialog:
// Radix Dialog sets pointer-events: none on body — you MUST add:
{createPortal(
  <div className="dropdown-menu pointer-events-auto">
    {items}
  </div>,
  document.body
)}

// Radix-native Select/Popover handle this automatically — no fix needed
```

## Zustand Store Pattern

```tsx
// Stores in ui/web/src/stores/
// Keep route-derived state OUT of stores
const useAgentStore = create<AgentState>((set) => ({
  agents: [],
  loading: false,
  fetchAgents: async () => {
    set({ loading: true });
    const data = await api.agents.list();
    set({ agents: data, loading: false });
  },
}));
```

## i18n — All 3 Locales Required

```tsx
// CORRECT — add key to en/vi/zh in same commit
// ui/web/src/i18n/locales/en/common.json
{ "agent.created": "Agent created" }

// ui/web/src/i18n/locales/vi/common.json
{ "agent.created": "Đã tạo agent" }

// ui/web/src/i18n/locales/zh/common.json
{ "agent.created": "代理已创建" }

// In component:
const { t } = useTranslation('common');
return <p>{t('agent.created')}</p>;
```

## Responsive Grid Pattern

```tsx
// CORRECT — mobile first
<div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">

// WRONG — breaks on mobile
<div className="grid grid-cols-3 gap-4">
```

## Web vs Desktop — Different Projects

| Concern | Web (`ui/web/`) | Desktop (`ui/desktop/frontend/`) |
|---|---|---|
| Framework | React Router 7 | No router (single page) |
| Animation | — | Framer Motion |
| Backend calls | HTTP/WS via api/ | Wails Go bindings |
| State | Zustand | Zustand |
| i18n | i18next namespaces | Embedded (no i18next) |
| Build | `pnpm build` in `ui/web/` | `wails build` in `ui/desktop/` |

Always confirm which target before implementing a UI feature.
