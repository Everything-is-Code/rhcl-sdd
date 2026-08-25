## Context

See [proposal.md](./proposal.md) for motivation. The frontend currently has no `components/` or `hooks/` directory — all sub-components are inlined inside page files. `AppState` is defined and managed in `App.tsx`'s `AppContent` and prop-drilled to 7 workflow routes. Three pages declare `setAppState` in Props but never use it.

**Sequencing**:
1. [PR #190](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/190) (backend Strategy+Registry, #40) merges first — backend-only, no frontend conflict, but touches `AGENTS.md`
2. **This change** (#41) — frontend refactor gate, no other frontend work until merged
3. [#171](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/171) — Backend error envelope (`@ServerExceptionMapper` + unified `{error, code, details}`)
4. [#149](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/149) — Policy conversions (builds on clean frontend + clean backend)

Related issues for later: [#172](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/172) (component tests), [#53](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/53) (YAML templates).

Key constraints:
- Stack: React 18, PatternFly 5, TypeScript strict, Vite, i18next (en/ja)
- No `@testing-library/react` — 0% component test coverage; logic/util tests only
- `sessionStorage` persistence via `appStateStorage.ts` with security invariants (I-2/I-3: never persist `accessToken`)
- CSS Modules co-located with pages (`*.module.css`)

## Goals / Non-Goals

**Goals:**
- Workflow pages (`ImportPage`, `HistoryPage`, `ConversionPage`, `ConnectionPage`, `APISelectionPage`, `YAMLViewerPage`) reduced to **<200 lines** as orchestrators
- `App.tsx` shell (masthead, sidebar, routes, footer) may remain up to **~250 lines** — task 1.4 already scoped this separately from page orchestrators
- Replace `AppState` prop drilling with React Context for all workflow routes
- Remove dead `setAppState` props from pages that never use them
- Establish `src/components/` with domain-grouped subfolders
- Preserve 100% of existing behavior — no visible changes to the user

**Non-Goals:**
- Full `@testing-library/react` coverage of every extracted component — tracked in [#172](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/172)
- Introducing state management libraries (Redux, Zustand)
- Backend error envelope unification (`@ServerExceptionMapper`, `{error, code, details}`) — next change
- Changing i18n key structure — keys stay in current locale files

## Decisions

### D1: React Context for AppState

**Decision**: `AppStateContext` with `React.createContext` + `useAppState()` custom hook.

**Why Context**: minimal step without new dependencies; `AppState` changes are infrequent (connect, select, convert). Prop drilling already causes the same re-render scope.

**Why not Redux/Zustand**: overkill for one state object; Context can be upgraded later if complexity grows.

### D2: Context provider placement

```
App → BrowserRouter → AppStateProvider → AppContent (layout + routes)
```

Provider inside `BrowserRouter` so future routing-aware state (e.g. redirect on disconnect) stays possible.

### D3: Component extraction targets

Extract components that are self-contained, have clear prop boundaries, and are >50 lines.

**From App.tsx (→ `src/components/`):**

| Component | Reason |
|-----------|--------|
| `LangSwitcher` | Self-contained, reusable |
| `RouteErrorBoundary` | Cross-cutting concern |

`RedHatLogo` + `RHHatIcon` + `Footer` stay in `App.tsx` — masthead-only, tightly coupled. **Not dead code** — `RedHatLogo` renders in `MastheadBrand`.

**From ImportPage.tsx (→ `src/components/import/`):**

| Component | Props |
|-----------|-------|
| `TestInfoPanel` | `testInfo`, `namespace` |
| `YamlDropzone` | `onFilesLoaded`, `disabled?` |
| `YamlDiffViewer` | `original`, `edits`, `onEditChange` |
| `ImportResultTable` | `results` |

`parseTestInfo()` → `importUtils.ts` (pure logic, testable without mounting).

**From HistoryPage.tsx (→ `src/components/history/`):**

| Component | Responsibility |
|-----------|----------------|
| `HistoryToolbar` | Header actions (reload, delete selected), select-all toolbar, pagination |
| `HistoryTable` | Expandable table rows, download per entry, failure details |
| `HistoryDeleteModal` | Bulk delete confirmation modal |

`formatDate`, `statusColor`, `sourceLabel` → `historyUtils.ts`.

**From ConversionPage.tsx (→ `src/components/conversion/`):**

| Component | Responsibility |
|-----------|----------------|
| `ConversionForm` | Backend settings, output settings (TLS, DNS, logging, anonymous, IP check), convert button |
| `ConversionResults` | Progress bar, per-service result list with success/failure icons |

`toKebabName()` → `conversionUtils.ts`. Page orchestrator owns `handleConvert` API call and `useAppState()` writes.

**From ConnectionPage.tsx (→ `src/components/connection/`):**

| Component | Responsibility |
|-----------|----------------|
| `ConnectionForm` | URL, token, tenant, namespace fields + test connection |
| `ClusterVersionsPanel` | Profile selector, version refresh, capability labels, OCP/Gateway API/Kuadrant/OSSM display |

`displayOrDash` stays in page or moves to `connectionUtils.ts`. `apiErrorMessage` moves to the shared `utils/apiError.ts` helper (see D7).

**From APISelectionPage.tsx (→ `src/components/api/`):**

| Component | Responsibility |
|-----------|----------------|
| `PolicyPanel` | Policy definitions display (already inline — extract as-is) |
| `ServiceToolbar` | Search input, pagination controls |
| `ServiceList` | Service DataList with selection |

**From YAMLViewerPage.tsx (→ `src/components/yaml/`):**

| Component | Responsibility |
|-----------|----------------|
| `YamlFileTabs` | Service selector + file tab navigation |
| `YamlEditorPanel` | Side-by-side original/edited view, edit mode toggle, reset, copy |

### D4: File organization

```
frontend/src/
  components/
    AppStateContext.tsx
    LangSwitcher.tsx
    RouteErrorBoundary.tsx
    import/
      TestInfoPanel.tsx, YamlDropzone.tsx, YamlDiffViewer.tsx, ImportResultTable.tsx
      importUtils.ts
    history/
      HistoryToolbar.tsx, HistoryTable.tsx, HistoryDeleteModal.tsx
      historyUtils.ts, historyLabels.tsx
    conversion/
      ConversionForm.tsx, ConversionResults.tsx,
      ConversionBackendSettings.tsx, ConversionOutputSettings.tsx, ConversionPolicySettings.tsx,
      conversionUtils.ts, conversionFormTypes.ts
    connection/
      ConnectionForm.tsx, ClusterVersionsPanel.tsx
    api/
      PolicyPanel.tsx, ServiceToolbar.tsx, ServiceList.tsx
    yaml/
      YamlFileTabs.tsx, YamlEditorPanel.tsx
```

Domain subfolders keep navigability as `components/` grows. `AppState` type lives in `AppStateContext.tsx` — not in `api/types.ts` (frontend UI concern, not API contract). Pages that today import `AppState` from `../App` switch to `../components/AppStateContext`.

### D5: CSS Modules

CSS Modules co-located with components in domain subfolders (e.g. `components/import/import.module.css`). Pages may import the same module when they share layout styles. Shared tokens remain in `styles/shared.module.css`.

### D6: Dead code removal

- `setAppState` from `ValidationPage`, `DownloadPage`, `CompatibilityPage` Props — confirmed unused
- After Context migration, remove `appState`/`setAppState` from all route declarations

`RedHatLogo` is **not** dead code — no removal.

### D7: Shared `apiErrorMessage()` helper + `catch (e: unknown)`

**Decision**: Create `frontend/src/utils/apiError.ts` with a single typed helper that consolidates the 6 duplicated error extraction patterns across pages.

```typescript
export function apiErrorMessage(e: unknown, fallback: string): string {
  if (e && typeof e === 'object') {
    const err = e as {
      message?: string;
      response?: { data?: { error?: string; message?: string } };
    };
    return err.response?.data?.error
      || err.response?.data?.message
      || err.message
      || fallback;
  }
  if (typeof e === 'string' && e.trim()) return e;
  return fallback;
}
```

**Current state (6 inconsistent variants):**
- `ConnectionPage` has a proper `apiErrorMessage()` function with typed narrowing
- `ConversionPage`: inline `e.response?.data?.error || e.message`
- `APISelectionPage`: inline `e.response?.data || e.message` (returns object, not string!)
- `ImportPage`: 3 different inline patterns across catch blocks
- `DownloadPage`: bare `e.message`
- `HistoryPage`: bare `e.message`

**Why now**: since we're touching every page for the Context migration and component split, adding the import is zero extra churn. This closes tech specs §7 recommendation #7.

**Fix `catch (e: any)` → `catch (e: unknown)`**: TypeScript strict mode should use `unknown`; the shared helper handles the narrowing. Every `catch` block in page files switches to `(e: unknown)` and calls `apiErrorMessage(e, t('page.fallbackKey'))`.

**Why not backend envelope too**: backend error shapes are inconsistent (3 different shapes across controllers), but fixing that requires `@ServerExceptionMapper` + custom exceptions + updating all controllers — tracked as [#171](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/171), next change after this one. The frontend helper already handles all 3 backend shapes so it's forward-compatible.

## Risks / Trade-offs

**[Re-render scope]** → Context changes re-render all consumers. Mitigation: same as current prop drilling — `AppState` changes are user-triggered only.

**[Large PR scope]** → Many files touched in one change. Mitigation: phased tasks (Context first, then pages one-by-one); each phase verified with `typecheck` + `test` before continuing.

**[Component test scope]** → Unit tests for extracted pure logic (`*Utils.ts`, `AppStateContext`) plus minimal RTL smoke for one representative component. Full component coverage deferred to #172.

**[CSS cross-directory imports]** → Resolved: CSS modules live under `components/<domain>/`.

## Migration Plan

Five phases, ordered by dependency:

1. **Phase 1 — Context + App cleanup**: `AppStateContext`, extract `LangSwitcher` + `RouteErrorBoundary`, migrate all workflow pages to `useAppState()`, remove dead props.
2. **Phase 2 — Error handling cleanup**: create shared `apiErrorMessage()` in `utils/apiError.ts`, replace all inline variants and `catch (e: any)` across all pages.
3. **Phase 3 — ImportPage split**: largest file first; establishes `components/import/` pattern.
4. **Phase 4 — Remaining large pages**: HistoryPage, ConversionPage, ConnectionPage, APISelectionPage, YAMLViewerPage — one page group per task batch, verify after each.
5. **Phase 5 — Full verification**: `typecheck` + `npm test` + `npm run lint` + `mvn verify` + manual smoke.

Rollback: revert the PR. No data migration, no API changes.
