# AI Code Review

_Disclaimer: AI-generated review. Human maintainer review required._

**Change:** cluster-detection-ux | **Issue:** [#228](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/228)  
**Branch:** `feature/228-cluster-detection-ux` (local uncommitted diff)

## Summary

Focused, spec-aligned change: adds `capabilities.clusterReachable`, gates probe-dependent compatibility warnings, and improves Connection page UX to separate OpenShift reachability from 3scale login. Backend and frontend tests updated appropriately. No conversion pipeline / #40 / #170 / #169 impact.

## Major

_None._

## Moderate

1. **Manual verification still open (task 4.2)** — Workshop scenario from #228 (`mvn quarkus:dev` without `oc login`, then after `oc login`) is not yet evidenced. Unit tests cover logic; recommend completing 4.2 before merge or noting in PR test plan.

2. **Profile `ocp-4.19` Kuadrant copy** — `getClusterConnectionUiState` returns `profile` for any profile source; `kuadrantDisplayI18nKey` shows `kuadrantAbsent` when `kuadrantPresent` is false (4.19 profile), not `kuadrantProfileAssumed`. Only 4.21 forces `kuadrantPresent: true`. Users on 4.19 profile may see "Operator not found on cluster" even though they chose a manual profile — slightly inconsistent with profile-assumption messaging. Consider profile-specific copy or documenting 4.19 behavior.

3. **No live E2E** — Connection UX change is UI-heavy; `/verify-fe` not run. Recommend before archive (see verify-report).

## Minor

1. **`shouldShowClusterVersionsCard`** — Always returns `true`; unused `_connected` parameter and wrapper call in `ConnectionPage` could be removed in a follow-up for clarity (function may be redundant).

2. **Backend compatibility strings** — `Cluster connection` message is hardcoded English in `CompatibilityService` (same as existing Kuadrant/CORS capability messages). Acceptable for consistency; frontend i18n does not apply to compatibility API items today.

3. **`clusterReachable = true` on profile** — Intentional per design (avoid false cluster-connection warning when user chose profile). Field name slightly overloads "reachable" vs "assumed" — document in PR if reviewers ask.

4. **Test gap** — No `ConnectionPage` test asserting `loadVersions` on mount without 3scale connect; covered indirectly via `shouldShowClusterVersionsCard` + panel smoke test.

## Nit

1. **`kuadrantDisplayI18nKey` return type** includes `connection.kuadrantDetected` but detected path renders raw version string in the panel — harmless, type could be narrowed later.

2. **`ClusterVersionsPanel.test.tsx`** — Single unreachable scenario; profile/reachable banners untested (acceptable for smoke scope).

## Spec compliance

| Area | Verdict |
|------|---------|
| `clusterReachable` contract | OK |
| Compatibility gating | OK |
| Connection page UX | OK (profile 4.19 Kuadrant copy — see Moderate #2) |
| i18n en/ja | OK |
| Out of scope (Option B/C) | Respected |

## Security

No secrets, tokens, or kubeconfig paths in new user-facing strings. `ClusterVersionService.sanitize` unchanged.

## Recommendation

**Approve with minor follow-ups** — merge-ready after task 4.2 manual check (or explicit maintainer sign-off). Run `/verify-fe` before archive.
