---
description: Frontend standards — React 18, PatternFly 5, Vite 4, TypeScript strict, Vitest (no component tests yet).
globs: ["../migration-toolkit-rhcl/frontend/**"]
alwaysApply: true
---

# Frontend Standards — Migration Toolkit

> Ground truth: `../migration-toolkit-rhcl/TECHNICAL_SPECIFICATIONS.md` §1, §3, §5.7–5.9  
> Detail: `../migration-toolkit-rhcl/.cursor/rules/frontend-standards.mdc`

## Stack

| Component | Choice |
|-----------|--------|
| UI | React 18, PatternFly 5 |
| Build | Vite 4, TypeScript 5 (`strict: true`) |
| Routing | react-router-dom 6 |
| i18n | i18next / react-i18next (`en`, `ja`) |
| HTTP | Axios (`frontend/src/api/client.ts`) |
| Tests | Vitest, ESLint 9 flat config |
| Node | **22** (matches `frontend/Dockerfile.ci`) |

## Structure (39 files — no `components/` or `hooks/`)

```
frontend/src/
  pages/     One PatternFly page per route; subcomponents co-located in same file
             Sibling *.ts for testable logic (compatibilityChecks.ts, clusterCapabilityUi.ts)
  api/       client.ts (Axios + domain APIs), types.ts (all interfaces)
  locales/   en.json, ja.json — smoke test enforces parity
  styles/    pfTokens.ts, shared.module.css
  utils/     appStateStorage.ts, fixHttpRoutePort.ts, timezone.ts
```

## API client

- Base URL: `import.meta.env.VITE_API_URL || ''` — **bake-time only**, never `REACT_APP_*`
- `Accept-Language` from `i18n.language` in request interceptor
- **No global Authorization** — pass `bearerHeaders(accessToken)` per call
- Domain APIs as plain objects: `connectionApi`, `conversionApi`, `historyApi`, …
- Types: `interface` for objects; `type` for unions; suffixes `Request`/`Response`/`Result`/`Page`

## Components & state

- `const XxxPage: React.FC<Props>` + `export default`
- Local `useState` — no Redux/Context/Zustand
- `AppState` prop-drilled from `App.tsx` to workflow pages
- `sessionStorage` via `appStateStorage.ts` — **never persist `accessToken`** (I-2/I-3 invariant)
- Multi-step flow: routes + `navigate()`, not PatternFly Wizard

## Forms & errors

- No zod/yup — manual `if` + boolean error state
- Prefer PatternFly `FormHelperText`; disable submit when invalid
- Error extraction (duplicated per page today):  
  `e.response?.data?.error || e.response?.data?.message || e.message || fallback`
- `RouteErrorBoundary` in `App.tsx`; `console.error` only in error boundaries

## Styling

PatternFly utilities + CSS Modules co-located with pages. Use `styles/pfTokens.ts` — no Tailwind/Sass/CSS-in-JS.

## i18n

- Keys: `{page}.{camelCase}`; prefixes `btnX`, `labelX`, `colX`, `errorX`, `ariaX`
- Every `en.json` key must exist in `ja.json`

## Testing

```bash
cd ../migration-toolkit-rhcl/frontend && npm install --legacy-peer-deps
cd ../migration-toolkit-rhcl/frontend && npm run typecheck
cd ../migration-toolkit-rhcl/frontend && npm test
```

- Vitest, `environment: 'node'` — **no `@testing-library/react`** today
- Co-located `*.test.ts` only (not `.test.tsx`)
- Covers utils, API client mocks, locale smoke — **0% React component coverage**

## Gaps (tracked in TECHNICAL_SPECIFICATIONS §7)

- Shared `apiErrorMessage()` helper not yet extracted
- Component tests not implemented
- History page pagination UI (#169)
