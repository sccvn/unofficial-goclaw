# GoClaw — TypeScript/React Code Review Checklist

Use this checklist when reviewing any TypeScript/React PR in `ui/web/src/` or `ui/desktop/frontend/src/`.

---

## Mobile UX

- [ ] **`h-dvh` not `h-screen`** — `h-screen` breaks on mobile browser chrome and virtual keyboards
- [ ] **Input font-size** — all `<input>`, `<textarea>`, `<select>` use `text-base md:text-sm` (≥16px on mobile prevents iOS Safari auto-zoom)
- [ ] **Touch targets** — icon buttons and interactive elements have ≥44px hit area on `@media (pointer: coarse)`
- [ ] **Tables scrollable** — all `<table>` elements wrapped in `<div className="overflow-x-auto">` with `min-w-[600px]` on table
- [ ] **Safe areas** — edge-anchored elements (app shell, sidebar, chat input, toasts) use `safe-top`, `safe-bottom`, `safe-left`, `safe-right` classes
- [ ] **Landscape** — top bars use `landscape-compact` to reduce padding in phone landscape mode

## Routing & State

- [ ] **`useParams()` as source of truth** — route params not duplicated into `useState`; no dual state that can race
- [ ] **Optional params** over two separate routes when a route has optional segments (e.g. `/chat/:sessionKey?`)
- [ ] **`ErrorBoundary` stable key** — uses `stableErrorBoundaryKey(pathname)` not `location.pathname`; wrapping `<Outlet>` must not remount on param changes
- [ ] **No `useEffect` → `navigate` → `setState` cycles** that cause UI flash

## Components

- [ ] **Dialogs** — `max-sm:inset-0` (full-screen mobile) + `sm:max-w-lg` (centered desktop); matches `ui/dialog.tsx` pattern
- [ ] **Portal dropdowns in dialogs** — `createPortal` dropdowns have `pointer-events-auto` class (Radix Dialog sets `pointer-events: none` on body)
- [ ] **Responsive grids** — `grid-cols-1 sm:grid-cols-2 lg:grid-cols-N`; no fixed `grid-cols-N` without mobile breakpoint
- [ ] **`overscroll-contain`** on scrollable areas to prevent background scroll

## TypeScript

- [ ] **No `any` types** without justification
- [ ] **API response types** defined in `ui/web/src/types/` and used in API adapters
- [ ] **Nullability handled** — optional chaining (`?.`) and nullish coalescing (`??`) where data may be absent
- [ ] **No non-null assertions (`!`)** on data from network responses

## i18n

- [ ] **All 3 locale files updated** — new string keys added to `en/`, `vi/`, `zh/` in same commit
- [ ] **No hardcoded English strings** in UI components — all user-visible text uses `t()` translation hook
- [ ] **Namespace matches file** — `useTranslation('common')` only accesses keys defined in `common.json`

## API Integration

- [ ] **Adapters in `ui/web/src/api/`** — no direct `fetch()` in components; go through typed adapter functions
- [ ] **Error states handled** — loading and error states shown to user, not silently swallowed
- [ ] **No sensitive data in URL params** — tokens, API keys, etc. are never passed as query strings

## Performance

- [ ] **No re-renders in tight loops** — memoize callbacks with `useCallback`, expensive computations with `useMemo` when appropriate
- [ ] **Lazy imports** for heavy components/pages: `const MyPage = lazy(() => import('./MyPage'))`
- [ ] **No `useEffect` with missing deps** — ESLint exhaustive-deps rule should pass

## Build & Quality

- [ ] **`pnpm build`** passes without TypeScript errors in the affected UI directory
- [ ] **No unused imports** — TypeScript `noUnusedLocals` should not flag new code
- [ ] **`pnpm lint`** passes (ESLint config in `eslint.config.js`)
- [ ] **Web vs Desktop scope** — changes in `ui/web/` do not accidentally modify `ui/desktop/frontend/` and vice versa
