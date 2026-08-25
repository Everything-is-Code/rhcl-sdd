## Why

Several frontend pages exceed 300–800 lines and mix multiple responsibilities (layout, state management, API calls, sub-components) within single files. `AppState` is prop-drilled from `App.tsx` through 7 routes, and 3 pages declare `setAppState` in their Props interface without ever using it. This accumulated debt makes every frontend change harder and riskier because it touches oversized files with high merge-conflict probability.

This refactor is a **gate**: no parallel frontend feature work (including policy conversion #149) starts until the component split and Context migration land. Splitting now establishes a `components/` layer and smaller page orchestrators that future work can extend safely.

GitHub issue: [#41](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/41)

## What Changes

- **Extract `AppStateContext`**: replace `appState`/`setAppState` prop drilling with a React Context provider and `useAppState()` hook
- **Split `App.tsx` (~381 lines)**: extract `LangSwitcher`, `RouteErrorBoundary`; slim `App.tsx` to routing + layout shell
- **Split `ImportPage.tsx` (~866 lines)**: extract `YamlDropzone`, `YamlDiffViewer`, `TestInfoPanel`, `ImportResultTable` into `src/components/import/`
- **Split remaining large pages** (all in this change):
  - `HistoryPage` (~452) → `HistoryToolbar`, `HistoryTable`, `HistoryDeleteModal`
  - `ConversionPage` (~566) → `ConversionForm`, `ConversionResults`
  - `ConnectionPage` (~483) → `ConnectionForm`, `ClusterVersionsPanel`
  - `APISelectionPage` (~392) → `ServiceToolbar`, `ServiceList`, `PolicyPanel`
  - `YAMLViewerPage` (~310) → `YamlFileTabs`, `YamlEditorPanel`
- **Remove dead code**: unused `setAppState` prop from `ValidationPage`, `DownloadPage`, `CompatibilityPage`
- **Extract shared `apiErrorMessage()` helper**: duplicated error extraction logic across 6 pages (`e.response?.data?.error || ...`) consolidated into `src/utils/apiError.ts`; fix all `catch (e: any)` → `catch (e: unknown)` with typed narrowing
- **Create `src/components/` directory**: domain-grouped subfolders (`import/`, `history/`, `conversion/`, `connection/`, `api/`, `yaml/`)

## Capabilities

### New Capabilities

_None — this is a pure internal refactor with no behavior changes._

### Modified Capabilities

_None — `skip_specs: true` is set. All externally observable behavior (routes, API calls, rendered UI, i18n) remains identical._

## Impact

- **Frontend only** — no backend, API, or database changes
- **Files modified**: `App.tsx`, all large pages listed above, plus `ValidationPage`, `DownloadPage`, `CompatibilityPage`, all pages with error handling
- **Files created**: `src/components/` tree (~20 new files — Context, layout helpers, page-specific sub-components) + `src/utils/apiError.ts`
- **i18n**: keys stay in `en.json`/`ja.json`; locale smoke test enforces parity
- **Testing**: `npm run typecheck` + `npm test` + `mvn verify` (Playwright E2E) + manual smoke of all routes
- **Sequencing**: [PR #190](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/190) (#40 backend refactor) merges first → **this change** → [#171](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/171) (backend error envelope) → #149 (policies). No parallel frontend work until this merges.
