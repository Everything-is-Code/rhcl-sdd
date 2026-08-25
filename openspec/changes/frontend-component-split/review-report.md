# AI Code Review

_Disclaimer: AI-generated review. Human maintainer review required._

**Change:** `frontend-component-split` | **Issue:** [#41](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/41)

**Scope:** `../migration-toolkit-rhcl/` — 10 modified page files + 25 new files under `frontend/src/components/` and `frontend/src/utils/apiError*`. ~2,843 lines removed from pages, net structural refactor only (no REST/API contract changes).

### Major

None identified.

### Moderate

1. **Layer inversion: `components/` importing from `pages/`** — Several extracted components import page-local modules (`supportedPolicies`, `clusterCapabilityUi`) and page CSS modules (`ImportPage.module.css`, etc.). This creates a dependency cycle risk (`pages → components → pages`). Consider moving shared helpers to `utils/` or `pages/shared/` and co-locating styles under `components/<area>/` in a follow-up.

2. **`ConversionForm.tsx` size (433 lines)** — Page orchestrators meet the < 200 line goal, but the largest extracted component remains a mini-page. Future split (e.g. output settings vs policy settings) would improve reviewability.

3. **Design drift: `handleConvert` location** — `design.md` places the convert API call in the page orchestrator; implementation keeps it inside `ConversionForm`. Behavior is preserved; structure differs from the written design.

4. **Git staging** — All new `components/` and `apiError*` files are untracked. PR must include them explicitly; `git diff main` alone understates the change.

### Minor

1. **`App.tsx` at 208 lines** — Task 5.5 target was < 200 (Phase 1 target < 250 was met earlier).

2. **`historyUtils.tsx` vs `historyUtils.ts`** — Task artifact names `.ts`; file is `.tsx` because `sourceLabel` returns JSX (correct TypeScript choice).

3. **ESLint warnings** — `react-hooks/exhaustive-deps` on `APISelectionPage` and `CompatibilityPage` (intentional mount-only effects, inherited pattern). `react-refresh/only-export-components` on `AppStateContext` (exports hook + provider).

4. **No component-level tests** — Only `apiError.test.ts` added; extracted UI components rely on E2E/manual coverage. Acceptable for pure-move refactor but increases regression risk for future edits.

### Nit

1. `importUtils.ts` — `no-regex-spaces` warning (fixable with `{2}` quantifier).

2. `ConversionForm` imports `loadSupportedPolicies` from `pages/` — rename/move would clarify boundaries.

## Spec / architecture alignment

- No backend changes; #40 Strategy/Registry (#190) and #170 `generateReadme` not touched.
- `AppStateProvider` placement matches design D2.
- `apiErrorMessage` centralization matches design D7.
- Prop drilling removed from workflow pages as specified.

## Recommendation

Approve after human review with awareness of layer-inversion follow-up. Do not auto-merge without CODEOWNERS and full file list in PR description.
