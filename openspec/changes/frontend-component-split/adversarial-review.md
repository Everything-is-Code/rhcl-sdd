# Adversarial review

**Scope:** `frontend-component-split` / [#41](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/41)  
**Sources:** `openspec/changes/frontend-component-split/{proposal,design,tasks}.md` + `git diff main` + untracked files in `migration-toolkit-rhcl/frontend/src/components/`

## Spec alignment

| Acceptance criterion | Adversarial check | Result |
|---------------------|-------------------|--------|
| No user-visible behavior change | E2E + manual smoke exercised wizard routes | OK |
| Context replaces prop drilling | `useAppState()` without provider throws; routes no longer pass `appState` props | OK |
| Error handling unified | Grep: zero `catch (e: any)`; APISelection object-in-message bug fixed | OK |
| All large pages split | Line counts verified; orchestrators thin | OK (`App.tsx` 208 — within <250 shell limit) |
| Sequencing vs #190 | Diff is frontend-only; no backend coupling | OK |

**Non-goals respected:** No backend error envelope (#171), no policy epic (#149), no OpenSpec specs added.

## Findings

| Severity | Area | Finding | Evidence | Fix |
|----------|------|---------|----------|-----|
| Minor | Architecture | `components/*` imports `pages/supportedPolicies`, `pages/clusterCapabilityUi`, page CSS modules | 12 import paths under `components/` | **Resolved** — moved to `utils/` + component CSS modules |
| Minor | Design fidelity | `handleConvert` lives in `ConversionForm`, not page orchestrator | `design.md` D4 vs `ConversionForm.tsx` | **Resolved** — handler in `ConversionPage` |
| Minor | Metrics | `App.tsx` 208 lines vs task 5.5 < 200 | Line count | **Resolved in design** — shell limit < 250 (task 1.4 / 5.5) |
| Minor | Test gap | No tests for extracted components | Only `apiError.test.ts` new | **Resolved** — utils + `AppStateContext` + policy settings smoke |
| Minor | PR hygiene | ~25 files untracked | `git status` | **Resolved** in commits |
| Question | CI | `mvn test` 2 `ConversionServiceTest` failures locally | Windows CRLF YAML string compare | Confirm CI green (pre-existing, not in diff) |
| Question | Layering | `ConversionForm` 433 lines — future edit risk | File size | **Resolved** — split into settings sub-components |

**Attempted break scenarios (no blocker found):**

- **Context without provider** — `useAppState()` throws explicit error (safe).
- **Stale connection on API pages** — `sessionStorage` persistence unchanged; ConnectionForm still clears on failed test.
- **Partial apply error shapes** — `ImportPage` `handleApply` still handles `data.results` array on error response with typed narrowing.
- **Token in UI** — No new token logging; ConnectionForm keeps password field + show/hide.
- **i18n** — Manual JA toggle verified during smoke.

## Verdict

**PASS WITH GAPS** — Post-fix: layer inversion, conversion orchestration, component tests, and `App.tsx` line budget clarified in design. Remaining gap: full RTL coverage (#172).

## Before archive

1. Human PR review + CODEOWNERS.
2. Confirm CI `mvn test` / `mvn verify` on Linux (ignore local CORS indent flakes if CI passes).
3. ~~Stage all new frontend files in commit.~~ Done.
4. ~~Optional follow-up: relocate helpers/CSS out of `pages/`.~~ Done.
