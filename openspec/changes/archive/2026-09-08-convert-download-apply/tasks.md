## 1. Workflow state — validation snapshot

- [x] 1.1 Extend `AppState` with `validationSnapshot: { fingerprint: string; results: Record<string, ValidationResult> } | null` in `AppStateContext.tsx` and `api/types.ts`; verify `cd frontend && npm run typecheck` passes
- [x] 1.2 Add `clearValidationSnapshotIfStale(fingerprint)` helper in `conversionWorkflowState.ts` with unit tests covering fingerprint match/mismatch; verify `npm test -- conversionWorkflowState.test.ts` passes
- [x] 1.3 Update `ValidationPage.tsx` to write `validationSnapshot` after validate using `conversionResultsFingerprint`; verify manual flow: run validate → snapshot populated in React state (or unit test if page test added in §5)

## 2. Shared cluster apply preparation

- [x] 2.1 Extract `buildApplyPayload(yamlFiles, namespace)` into `frontend/src/utils/clusterApply.ts` (normalizeApiVersions, namespace substitution, external-backend `fixHttpRoutePort`); verify unit tests in `clusterApply.test.ts` cover normalization and placeholder detection helper
- [x] 2.2 Refactor `ImportPage.tsx` to use `buildApplyPayload` before `applyApi.apply`; verify existing `ImportPage.test.tsx` still passes (`cd frontend && npm test -- ImportPage.test.tsx`)

## 3. Apply guardrails helper

- [x] 3.1 Add `getApplyGuardrails(result, appState)` in `frontend/src/utils/applyGuardrails.ts` returning `{ enabled, reasonKey }` for clusterReachable, validation snapshot, ERROR items, REPLACE_ME, empty yamlFiles; verify unit tests for each disabled reason
- [x] 3.2 Wire stale snapshot clearing via `clearValidationSnapshotIfStale` on Download (and Validate) mount; verify unit test: changed fingerprint clears snapshot

## 4. Download page — apply UI

- [x] 4.1 Create `ApplyConfirmModal.tsx` (namespace, file count, packageName, RBAC note); verify component renders confirm/cancel in a minimal test or Download page test
- [x] 4.2 Update `DownloadPage.tsx`: per-row **Apply to cluster**, unreachable banner, namespace display, guardrail tooltips/disabled state, confirm modal, inline `ImportResultTable` after apply; verify apply calls `applyApi.apply(ns, files, 'CONVERT', packageName)` with workflow YAML
- [x] 4.3 Update Download "Next steps" copy to mention direct apply option; verify strings present in UI via test or snapshot

## 5. i18n

- [x] 5.1 Add `download.apply*`, `download.applyDisabled*`, `download.applyConfirm*` keys to `en.json` and `ja.json`; verify `npm test -- locales.smoke.test.ts` passes

## 6. Tests

- [x] 6.1 Add `DownloadPage.test.tsx` with mocked `applyApi` and `AppStateProvider`: disabled when `clusterReachable: false`, disabled when validation missing/ERROR/REPLACE_ME, happy path confirm → apply → result table; verify `npm test -- DownloadPage.test.tsx` passes
- [x] 6.2 Run `cd frontend && npm test && npm run typecheck` — all green

## 7. Documentation

- [x] 7.1 Update `migration-toolkit-rhcl/documentation/user-guide.md` §5–8 to document apply-from-Download without ZIP import; verify section mentions guardrails (validate, REPLACE_ME, cluster reachable)
- [x] 7.2 Link PR to [#313](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/313) and note `source: CONVERT` history behavior

## 8. Verification

- [x] 8.1 Manual check (cluster reachable): Convert → YAML edit → Validate → Download → Apply → History shows CONVERT entry; verify partial failure shows per-file errors in result table
- [x] 8.2 Manual check (cluster unreachable): Apply disabled with actionable banner; verify no apply request sent
