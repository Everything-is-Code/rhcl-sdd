# Verify Report — cluster-detection-ux

**Issue:** [#228](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/228)  
**Branch:** `feature/228-cluster-detection-ux` (uncommitted local diff)  
**Date:** 2026-08-27

## Scope

| Flag | Value |
|------|-------|
| `BACKEND_TOUCHED` | yes — `ClusterCapabilities`, `ClusterVersionService`, `CompatibilityService`, tests |
| `FRONTEND_TOUCHED` | yes — `ClusterVersionsPanel`, `ConnectionPage`, `clusterCapabilityUi`, i18n, tests |

## OpenSpec

- `openspec validate cluster-detection-ux --strict` — **PASS**
- Planning artifacts — **complete** (proposal, spec, design, tasks)

## Tasks (`tasks.md`)

| Task | Status |
|------|--------|
| 1.1–1.3 Backend `clusterReachable` | done |
| 2.1–2.3 Compatibility gating | done |
| 3.1–3.4 Frontend Connection UX | done |
| 4.1 Full test suites | done |
| **4.2 Manual workshop check** | **open** |
| 4.3 api-spec.yml | done (endpoint not documented in api-spec; N/A) |

## Test results

| Command | Result |
|---------|--------|
| `cd backend && mvn test` | **PASS** |
| `cd frontend && npm run typecheck` | **PASS** |
| `cd frontend && npm test` | **PASS** (76 tests) |

## Frontend E2E

- `verify-fe-report.md` — **not present**
- **Follow-up required:** run **`/verify-fe`** before archive (UI-heavy Connection page change). Not blocking PR if unit/smoke tests suffice for maintainer review.

## Spec compliance checklist

| Scenario | Evidence |
|----------|----------|
| Detected → `clusterReachable: true` | `ClusterVersionService.detectOrDefault`, `ClusterVersionServiceTest.resolve_whenSmcpForbidden_*` |
| Soft-fail → `clusterReachable: false`, `source: default` | `softFailDefault`, `resolve_whenClientNull_*` |
| Profile override unchanged assumptions | `applyProfile` + existing profile tests |
| Unreachable: no `kuadrantPresent` / `corsNative` / `ossmMatchesOcp` capability warnings | `CompatibilityServiceTest.check_unreachableCluster_*` |
| Unreachable: Cluster connection WARNING | `checkClusterConnectionWarning`, test above |
| Reachable + missing Kuadrant: existing warning | `check_missingKuadrant_warnsWithoutBlocking` (+ `clusterReachable: true`) |
| Reachable + Kuadrant present: no warning | `check_kuadrantPresent_noKuadrantWarning` |
| Panel visible without 3scale connect | `shouldShowClusterVersionsCard()` always true; `ConnectionPage` mount load |
| Unreachable banner (warning) + oc login guidance | `ClusterVersionsPanel` + `ClusterVersionsPanel.test.tsx` |
| Kuadrant not bare em dash when unreachable | `kuadrantUnreachable` i18n + `formatKuadrantValue` |
| Kuadrant absent when reachable | `kuadrantAbsent` i18n |
| Fallback OCP/GAPI labeled | `versionFallbackDefault` + `formatVersionValue` |
| Profile labeling | `clusterProfileTitle` / `clusterProfileBody` alerts |
| en + ja i18n parity | `locales.smoke.test.ts`, `clusterCapabilityUi.test.ts` |

## Outcome

**CONDITIONAL PASS**

Implementation matches spec and unit tests are green. **Do not archive** until:

1. Task **4.2** manual workshop verification (`quarkus:dev` without/with `oc login`).
2. **`/verify-fe`** (recommended) for Connection page live check.

## Follow-ups

| Priority | Item |
|----------|------|
| Blocking (archive) | 4.2 manual check |
| Recommended | `/verify-fe` |
| PR only | Link #228 in PR description |
