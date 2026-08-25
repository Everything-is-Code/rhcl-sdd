---
description: Frontend standards — React 18, PatternFly 5, Vite, TypeScript, react-i18next.
globs: ["../migration-toolkit-rhcl/frontend/**"]
alwaysApply: true
---

# Frontend Standards — Migration Toolkit

## Stack

| Component | Choice |
|-----------|--------|
| UI | React 18, PatternFly 5 |
| Build | Vite, TypeScript |
| i18n | react-i18next (`en`, `ja`) |
| HTTP | Axios via `frontend/src/api/client.ts` |
| Tests | Vitest / npm test |

## API Client

- Base URL: `import.meta.env.VITE_API_URL` (bake-time, not runtime)
- Local: `VITE_API_URL=http://localhost:8080 npm run dev`
- Cluster: set at `vite build` in `deploy/install.sh`
- Sends `Accept-Language` matching UI locale
- **Do not** use `REACT_APP_*` env vars

## Structure

```
frontend/src/
  api/          client.ts, API modules
  components/   PatternFly-based UI
  pages/        Route-level views (History, Convert, …)
  locales/      en.json, ja.json
```

## UI Guidelines

- PatternFly components and layout patterns consistently
- Masthead language toggle (EN / JA)
- Loading and error states for all async 3scale/cluster calls
- History page: backend supports pagination — expose it in UI when extending (#169)

## Testing

```bash
cd ../migration-toolkit-rhcl/frontend && npm run typecheck
cd ../migration-toolkit-rhcl/frontend && npm test
```

Install: `npm install --legacy-peer-deps`

## TypeScript

- Strict typing for API responses and form state
- Prefer explicit types over `any` for 3scale/K8s DTO shapes

## i18n

- Default language: English
- New user-facing strings: add to both `en.json` and `ja.json`
- Backend messages: parallel keys in `messages_en.properties` / `messages_ja.properties`
