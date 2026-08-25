## 1. Phase 1 — AppStateContext + App.tsx cleanup

- [x] 1.1 Create `frontend/src/components/AppStateContext.tsx` with `AppState` interface, `React.createContext`, `AppStateProvider` (owns `useState<AppState>` + `sessionStorage` persistence via `appStateStorage.ts`), and `useAppState()` hook. Verify: `npm run typecheck` passes.
- [x] 1.2 Create `frontend/src/components/LangSwitcher.tsx` — move from `App.tsx` with `useTranslation`, `i18n.changeLanguage`, and `App.module.css` style references. Verify: `npm run typecheck` passes.
- [x] 1.3 Create `frontend/src/components/RouteErrorBoundary.tsx` — move class component from `App.tsx`. Verify: `npm run typecheck` passes.
- [x] 1.4 Refactor `App.tsx`: import extracted components, wrap `AppContent` in `AppStateProvider` inside `BrowserRouter`, remove inlined `LangSwitcher`, `RouteErrorBoundary`, and `AppState` type. Keep `RedHatLogo`, `RHHatIcon`, `Footer` inline. Verify: `App.tsx` < 250 lines, `npm run typecheck` passes.
- [x] 1.5 Migrate all workflow pages to `useAppState()`: `ConnectionPage`, `APISelectionPage`, `CompatibilityPage`, `ConversionPage`, `YAMLViewerPage`, `ValidationPage`, `DownloadPage` — remove `appState`/`setAppState` from Props, update `AppState` imports from `../App` to `../components/AppStateContext`. Update route declarations to stop passing props. Verify: `npm run typecheck` passes.
- [x] 1.6 Confirm dead `setAppState` removed from all page Props interfaces. Verify: grep `setAppState` in `frontend/src/pages/` returns zero Props-interface matches.
- [x] 1.7 Run `npm run typecheck` + `npm test`. Verify: zero errors.

## 2. Phase 2 — Error handling cleanup

- [x] 2.1 Create `frontend/src/utils/apiError.ts` with shared `apiErrorMessage(e: unknown, fallback: string): string` helper that handles all 3 backend response shapes (`{error}`, `{message}`, `{success, message}`) plus bare `Error.message` and string errors. Add unit test `apiError.test.ts` covering each shape. Verify: `npm run typecheck` + `npm test` passes.
- [x] 2.2 Replace all inline error extraction in `ConnectionPage` — remove the local `apiErrorMessage` function, import from `../utils/apiError`. Fix all `catch (e: any)` → `catch (e: unknown)`. Verify: `npm run typecheck` passes.
- [x] 2.3 Replace inline error extraction in `ConversionPage` — replace `e.response?.data?.error || e.message` with `apiErrorMessage()`. Fix `catch (e: any)` → `catch (e: unknown)`. Verify: `npm run typecheck` passes.
- [x] 2.4 Replace inline error extraction in `APISelectionPage` — fix `e.response?.data || e.message` (returns object!) with `apiErrorMessage()`. Fix `catch (e: any)` → `catch (e: unknown)`. Verify: `npm run typecheck` passes.
- [x] 2.5 Replace inline error extraction in `ImportPage` — 3 different inline patterns across catch blocks, all replaced with `apiErrorMessage()`. Fix `catch (e: any)` → `catch (e: unknown)`. Verify: `npm run typecheck` passes.
- [x] 2.6 Replace inline error extraction in `DownloadPage` and `HistoryPage` — bare `e.message` replaced with `apiErrorMessage()`. Fix `catch (e: any)` → `catch (e: unknown)`. Verify: `npm run typecheck` passes.
- [x] 2.7 Audit remaining pages (`ValidationPage`, `CompatibilityPage`, `YAMLViewerPage`, `SettingsPage`) for any `catch (e: any)` or inline error patterns and fix. Verify: grep for `catch.*any` in `frontend/src/pages/` returns zero matches.
- [x] 2.8 Run `npm run typecheck` + `npm test`. Verify: zero errors, new `apiError.test.ts` passes.

## 3. Phase 3 — ImportPage split

- [x] 3.1 Create `frontend/src/components/import/importUtils.ts` — move `parseTestInfo()` and types (`TestInfo`, `RouteInfo`, `AuthInfo`, `YamlFile`, `EditMap`, `ApplyResult`). Verify: `npm run typecheck` passes.
- [x] 3.2 Create `frontend/src/components/import/TestInfoPanel.tsx` — gateway URL polling, auth info, curl builder. Import styles from `../../pages/ImportPage.module.css`. Verify: `npm run typecheck` passes.
- [x] 3.3 Create `frontend/src/components/import/YamlDropzone.tsx` — file upload/drag-drop with `onFilesLoaded` prop. Verify: `npm run typecheck` passes.
- [x] 3.4 Create `frontend/src/components/import/YamlDiffViewer.tsx` — side-by-side YAML diff with tab navigation. Verify: `npm run typecheck` passes.
- [x] 3.5 Create `frontend/src/components/import/ImportResultTable.tsx` — apply results table. Verify: `npm run typecheck` passes.
- [x] 3.6 Refactor `ImportPage.tsx` to compose extracted components. Verify: `ImportPage.tsx` < 200 lines, `npm run typecheck` passes.
- [x] 3.7 Run `npm run typecheck` + `npm test`. Verify: zero errors.

## 4. Phase 4 — Remaining large pages

### 4A — HistoryPage

- [x] 4.1 Create `frontend/src/components/history/historyUtils.ts` — move `formatDate`, `statusColor`, `sourceLabel`. Verify: `npm run typecheck` passes.
- [x] 4.2 Create `frontend/src/components/history/HistoryDeleteModal.tsx` — bulk delete confirmation modal. Verify: `npm run typecheck` passes.
- [x] 4.3 Create `frontend/src/components/history/HistoryToolbar.tsx` — header actions, select-all toolbar, pagination. Verify: `npm run typecheck` passes.
- [x] 4.4 Create `frontend/src/components/history/HistoryTable.tsx` — expandable table rows, per-entry download, failure details. Verify: `npm run typecheck` passes.
- [x] 4.5 Refactor `HistoryPage.tsx` to orchestrate history components. Verify: `HistoryPage.tsx` < 200 lines, `npm run typecheck` passes.

### 4B — ConversionPage

- [x] 4.6 Create `frontend/src/components/conversion/conversionUtils.ts` — move `toKebabName()`. Verify: `npm run typecheck` passes.
- [x] 4.7 Create `frontend/src/components/conversion/ConversionForm.tsx` — backend settings, output settings (TLS, DNS, logging, anonymous, IP check), convert button. Verify: `npm run typecheck` passes.
- [x] 4.8 Create `frontend/src/components/conversion/ConversionResults.tsx` — progress bar and per-service result list. Verify: `npm run typecheck` passes.
- [x] 4.9 Refactor `ConversionPage.tsx` to orchestrate form + results via `useAppState()`. Verify: `ConversionPage.tsx` < 200 lines, `npm run typecheck` passes.

### 4C — ConnectionPage

- [x] 4.10 Create `frontend/src/components/connection/ConnectionForm.tsx` — URL, token, tenant, namespace, test connection. Verify: `npm run typecheck` passes.
- [x] 4.11 Create `frontend/src/components/connection/ClusterVersionsPanel.tsx` — profile selector, version refresh, capability labels, version description list. Verify: `npm run typecheck` passes.
- [x] 4.12 Refactor `ConnectionPage.tsx` to orchestrate form + versions panel via `useAppState()`. Verify: `ConnectionPage.tsx` < 200 lines, `npm run typecheck` passes.

### 4D — APISelectionPage

- [x] 4.13 Create `frontend/src/components/api/PolicyPanel.tsx` — move existing inline `PolicyPanel`. Verify: `npm run typecheck` passes.
- [x] 4.14 Create `frontend/src/components/api/ServiceToolbar.tsx` — search input and pagination controls. Verify: `npm run typecheck` passes.
- [x] 4.15 Create `frontend/src/components/api/ServiceList.tsx` — service DataList with selection. Verify: `npm run typecheck` passes.
- [x] 4.16 Refactor `APISelectionPage.tsx` to orchestrate toolbar + list + policy panel via `useAppState()`. Verify: `APISelectionPage.tsx` < 200 lines, `npm run typecheck` passes.

### 4E — YAMLViewerPage

- [x] 4.17 Create `frontend/src/components/yaml/YamlFileTabs.tsx` — service selector and file tab navigation. Verify: `npm run typecheck` passes.
- [x] 4.18 Create `frontend/src/components/yaml/YamlEditorPanel.tsx` — original/edited view, edit mode, reset, copy. Verify: `npm run typecheck` passes.
- [x] 4.19 Refactor `YAMLViewerPage.tsx` to orchestrate tabs + editor via `useAppState()`. Verify: `YAMLViewerPage.tsx` < 200 lines, `npm run typecheck` passes.

- [x] 4.20 Run `npm run typecheck` + `npm test` after all Phase 4 splits. Verify: zero errors.

## 5. Verification

- [x] 5.1 Run frontend checks: `cd frontend && npm run typecheck && npm test && npm run lint`. Verify: all exit 0.
- [x] 5.2 Run backend + Playwright E2E: `cd backend && mvn verify`. Verify: `PlaywrightE2EIT` passes.
- [x] 5.3 Manual smoke: start backend + frontend, navigate all 11 routes and exercise connection → services → compatibility → convert → yaml → validate → download flow plus import, history, settings. Verify: every page renders, language toggle works, connection state persists across navigation.
- [x] 5.4 Verify component tree exists per design D4 (`components/` with `import/`, `history/`, `conversion/`, `connection/`, `api/`, `yaml/` subfolders). Verify: directory listing matches design.
- [x] 5.5 Verify line counts: `ImportPage.tsx`, `HistoryPage.tsx`, `ConversionPage.tsx`, `ConnectionPage.tsx`, `APISelectionPage.tsx`, `YAMLViewerPage.tsx` all < 200 lines; `App.tsx` < 250 lines (shell orchestrator). Verify: line count check on each file.
- [x] 5.8 Add tests for extracted component logic: `importUtils.test.ts`, `conversionUtils.test.ts`, `historyUtils.test.ts`, `AppStateContext.test.tsx`, `ConversionPolicySettings.test.tsx`. Verify: `npm test` passes (52+ tests).
- [x] 5.6 Verify zero `catch (e: any)` remain in `frontend/src/`. Verify: grep returns zero matches.
- [x] 5.7 Verify zero inline error extraction duplicates remain — all pages use shared `apiErrorMessage()` from `utils/apiError.ts`. Verify: grep for `\.response\?\.data\?\.error` in pages returns zero matches outside of `apiError.ts`.
