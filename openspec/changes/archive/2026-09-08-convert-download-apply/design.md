## Context

See [proposal.md](./proposal.md). The conversion happy path today ends at ZIP download; Import already calls `POST /api/apply` with `source: IMPORT`. The backend `ApplyController` supports `source: CONVERT`, history recording, SSA, namespace ensure, and RBAC bootstrap—no new server work is required.

Relevant product code (`migration-toolkit-rhcl/frontend`):

- `DownloadPage.tsx` — ZIP download only today
- `ImportPage.tsx` — apply orchestration, `normalizeApiVersions`, `fixHttpRoutePort`, `ImportResultTable`
- `ValidationPage.tsx` — validation in **local state**; Next to Download only requires validate to have run once (not pass)
- `YAMLViewerPage.tsx` — merges edits into `appState.conversionResults` via `saveAndNavigate`
- `AppStateContext.tsx` — `namespace`, `conversionResults`, `clusterVersions`
- `conversionWorkflowState.ts` — `conversionResultsFingerprint` for stale-result detection (#229)
- `utils/clusterCapabilityUi.ts` — reachability helpers from cluster-detection change

Backend note: `ValidationService` flags `REPLACE_ME` as **WARNING**, not ERROR—apply guard for placeholders is a separate client scan.

## Goals / Non-Goals

**Goals:**

- Wire **Apply to cluster** on Download using existing `applyApi.apply(..., 'CONVERT', ...)`.
- Enforce guardrails from [#313](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/313): `clusterReachable`, validation snapshot without ERROR, no `REPLACE_ME`, non-empty YAML.
- Persist validation snapshot in workflow state with fingerprint invalidation.
- Confirmation modal before apply; inline `ImportResultTable` for results.
- Parity with Import apply prep (normalization, external-backend port fix) via shared helper.
- Unit tests + i18n + user guide update.

**Non-Goals:**

- New backend apply endpoint or semantics.
- Bulk apply across multiple services.
- `TestInfoPanel` / post-apply curl smoke test (Import-only).
- Multi-cluster apply.
- Auto-apply after validation.
- Tightening Validate → Download navigation to block on INVALID (ZIP download may still proceed with warnings/errors unless product decides otherwise later).

## Decisions

### 1. New workflow fields in `AppState` (validation snapshot)

**Choice:** Extend `AppState` with:

```text
validationSnapshot: {
  fingerprint: string;  // conversionResultsFingerprint(...)
  results: Record<serviceId, ValidationResult>;
} | null
```

Written by `ValidationPage` after a successful validate run. Cleared when fingerprint mismatches (effect in Download/Validate or shared hook).

**Rationale:** Download must enforce guardrails without re-calling `/api/validate` on every visit. Fingerprint reuse aligns with #229 stale-result pattern.

**Alternative rejected:** Keep validation only in Validate page local state — Download cannot enforce ERROR guard without duplicating validate API calls.

### 2. REPLACE_ME guard: client scan, not validation ERROR

**Choice:** Pure function `yamlFilesContainPlaceholder(files, 'REPLACE_ME')` on workflow YAML; disable apply when true.

**Rationale:** Backend validation emits WARNING for placeholders; `valid` may still be `true`. Issue AC requires explicit block.

### 3. Slim apply UI on Download (do not reuse full `NamespaceFormCard`)

**Choice:** Show read-only namespace from `appState.namespace` with helper text linking to Connection for changes. Per-row **Apply to cluster** + existing **Download ZIP**. Optional compact namespace row above the package list if multiple services ever appear.

**Rationale:** `NamespaceFormCard` bundles Import-only concerns (editable package name, port-fix notice, paired Download button). Convert path already has `packageName` on each result row.

**Alternative rejected:** Reuse `NamespaceFormCard` wholesale — wrong shape and duplicate package-name editing.

### 4. Shared apply preparation helper

**Choice:** Extract `buildApplyPayload(yamlFiles, namespace, options?)` (or `useClusterApply`) from Import logic into e.g. `frontend/src/utils/clusterApply.ts`:

- `normalizeApiVersions` on each file
- Replace `namespace:` lines with target namespace (Import already does this on "Apply namespace")
- `fixHttpRoutePort` when external backend detected (`detectExternalBackend`)

Download and Import both call this before `applyApi.apply`.

**Rationale:** External-backend conversions need the same HTTPRoute port fix Import applies; avoid divergent cluster behavior.

### 5. Confirmation modal (new component)

**Choice:** `ApplyConfirmModal.tsx` (PatternFly `Modal`, small variant)—pattern from `HistoryDeleteModal`. Props: `isOpen`, `namespace`, `fileCount`, `packageName`, `onConfirm`, `onClose`.

**Rationale:** Issue requires confirmation; Import has none today—keep scope to Download only.

### 6. Cluster reachability on Download

**Choice:** Disable apply when `appState.clusterVersions?.capabilities?.clusterReachable !== true`. Show `Alert` banner using existing i18n keys where possible (`connection.*` / cluster-detection copy) plus Download-specific apply hint.

**Note:** `clusterVersions` loads on Connection page mount. If user refreshes on `/download` without visiting Connection, `clusterVersions` is null → treat as unreachable (safe default). Optional follow-up: load versions on Download mount—out of scope unless needed during apply.

### 7. Results display

**Choice:** Reuse `ImportResultTable` with existing `import.*` result strings (or add thin `download.applyResult*` aliases if copy must differ). No `TestInfoPanel`.

### 8. i18n and docs

**Choice:** Add `download.apply*`, `download.applyDisabled*`, `download.applyConfirm*` keys in `en.json` and `ja.json`. Update `documentation/user-guide.md` §5–8 to document direct apply from Download as an alternative to ZIP → Import.

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| Validation snapshot stale after YAML edit without re-validate | Fingerprint includes yaml content hash or invalidate snapshot when `conversionResults` yaml changes after validate timestamp |
| User edits YAML in viewer then navigates via sidebar without save | Pre-existing; apply uses `conversionResults` only—document in user guide; optional future: auto-save on route change |
| `clusterVersions` null after refresh blocks apply | Banner explains visiting Connection to refresh; optional lazy load in follow-up |
| Profile mode `clusterReachable: true` without live probe | Same as Import—apply may fail at runtime; cluster errors surface in result table |
| Validation WARNING-only packages allowed to apply | Per spec; cluster may reject unknown CRDs—result table shows failures |

## Migration Plan

- Frontend-only deploy with backend unchanged.
- No data migration.
- Rollback: revert frontend; Download returns to ZIP-only behavior.

## Open Questions

_(none — scope and guardrail choices resolved in exploration and recorded above)_
