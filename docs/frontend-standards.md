---
description: Frontend standards — React 18, PatternFly 5, Vite 4, TypeScript strict, Vitest + minimal RTL.
globs: ["../migration-toolkit-rhcl/frontend/**"]
alwaysApply: true
---

# Frontend Standards — Migration Toolkit

> Ground truth: [technical-specifications.md](./technical-specifications.md)  
> Detail: `../migration-toolkit-rhcl/.cursor/rules/frontend-standards.mdc`

## Stack

| Component | Choice |
|-----------|--------|
| UI | React 18, PatternFly 5 |
| Build | Vite 4, TypeScript 5 (`strict: true`) |
| Routing | react-router-dom 6 |
| i18n | i18next / react-i18next (`en`, `ja`) |
| HTTP | Axios (`frontend/src/api/client.ts`) |
| Tests | Vitest, ESLint 9 flat config, `@testing-library/react` + jsdom (smoke) |
| Node | **22** (matches `frontend/Dockerfile.ci`) |

## Structure

```
frontend/src/
  components/   Domain-grouped UI + AppStateContext, LangSwitcher, RouteErrorBoundary
                import/, history/, conversion/, connection/, api/, yaml/
  pages/        Thin orchestrators per route (<200 lines); API calls + useAppState writes
  api/          client.ts (Axios + domain APIs), types.ts (API contracts only)
  locales/      en.json, ja.json — smoke test enforces parity
  styles/       pfTokens.ts, shared.module.css
  utils/        apiError.ts, appStateStorage.ts, supportedPolicies.ts, clusterCapabilityUi.ts, …
  test/         Vitest setup (jest-dom)
```

Page-specific logic without JSX stays in `components/<domain>/*Utils.ts` or `pages/*.ts` (`compatibilityChecks.ts`). CSS Modules co-locate with components (`components/import/import.module.css`, etc.).

## API client

- Base URL: `import.meta.env.VITE_API_URL || ''` — **bake-time only**, never `REACT_APP_*`
- `Accept-Language` from `i18n.language` in request interceptor
- **No global Authorization** — pass `bearerHeaders(accessToken)` per call
- Domain APIs as plain objects: `connectionApi`, `conversionApi`, `historyApi`, …
- Types: `interface` for objects; `type` for unions; suffixes `Request`/`Response`/`Result`/`Page`

## Components & state

- `const XxxPage: React.FC` orchestrates; subcomponents live under `components/<domain>/`
- **`AppStateContext`** + `useAppState()` — workflow state; provider in `App.tsx` inside `BrowserRouter`
- `AppState` type in `components/AppStateContext.tsx` (not `api/types.ts`)
- `sessionStorage` via `appStateStorage.ts` — **never persist `accessToken`** (I-2/I-3 invariant)
- Page orchestrators own API calls and `setAppState`; presentational components receive props/callbacks
- Multi-step flow: routes + `navigate()`, not PatternFly Wizard

## Forms & errors

- No zod/yup — manual `if` + boolean error state
- Prefer PatternFly `FormHelperText`; disable submit when invalid
- **`apiErrorMessage(e, fallback)`** in `utils/apiError.ts` — all pages use `catch (e: unknown)`
- `RouteErrorBoundary` in `components/RouteErrorBoundary.tsx`; `console.error` only in error boundaries

## Styling

PatternFly utilities + CSS Modules co-located with components. Use `styles/pfTokens.ts` — no Tailwind/Sass/CSS-in-JS.

## i18n

- Keys: `{page}.{camelCase}`; prefixes `btnX`, `labelX`, `colX`, `errorX`, `ariaX`
- Every `en.json` key must exist in `ja.json`

## Testing

```bash
cd ../migration-toolkit-rhcl/frontend && npm install --legacy-peer-deps
cd ../migration-toolkit-rhcl/frontend && npm run typecheck
cd ../migration-toolkit-rhcl/frontend && npm test
```

- Vitest: default `environment: 'node'` for `*.test.ts`; `jsdom` for `*.test.tsx` (`vitest.config.ts`)
- Co-located `*.test.ts` / `*.test.tsx` next to source
- Utils + API mocks + locale smoke + minimal RTL (`AppStateContext`, representative components)
- Full component coverage tracked in [#172](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/172)

## Gaps (tracked in TECHNICAL_SPECIFICATIONS §7)

- History page pagination UI ([#169](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/169))
- Full `@testing-library/react` coverage ([#172](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/172))
