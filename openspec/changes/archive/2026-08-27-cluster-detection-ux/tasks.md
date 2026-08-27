## 1. Backend — `clusterReachable` contract

- [x] 1.1 Add `clusterReachable` boolean to `ClusterCapabilities.java` and `frontend/src/api/types.ts`; verify TypeScript compiles (`cd frontend && npm run typecheck`)
- [x] 1.2 Set `clusterReachable = true` in `detectOrDefault` success path and `false` in `softFailDefault`; verify `ClusterVersionServiceTest` soft-fail and detected cases assert the flag
- [x] 1.3 Leave `applyProfile` behavior unchanged; verify profile resolve tests still pass (`cd backend && mvn test -Dtest=ClusterVersionServiceTest`)

## 2. Backend — compatibility gating

- [x] 2.1 Add failing tests in `CompatibilityServiceTest`: unreachable + JWT → cluster connection WARNING, no `kuadrantPresent`; unreachable + cors → no `corsNative` warning; reachable + no kuadrant → existing kuadrant warning preserved
- [x] 2.2 Implement gating in `CompatibilityService`: skip probe-dependent capability warnings when `!clusterReachable`; add `checkClusterConnectionWarning()` with `capability: clusterReachable`; verify new tests pass (`mvn test -Dtest=CompatibilityServiceTest`)
- [x] 2.3 Smoke existing controller tests that mock compatibility (`ExportControllerTest`, `ConversionControllerTest`) — verify `mvn test` green for controller package

## 3. Frontend — Connection page UX

- [x] 3.1 Add `getClusterConnectionUiState()` and Kuadrant display helpers in `clusterCapabilityUi.ts` with unit tests in `clusterCapabilityUi.test.ts`
- [x] 3.2 Update `shouldShowClusterVersionsCard` to always show panel; load versions on `ConnectionPage` mount unconditionally; verify panel renders without 3scale connect in unit/smoke test
- [x] 3.3 Refactor `ClusterVersionsPanel`: warning banner when unreachable, profile banner when `source=profile`, fallback labels for default OCP/GAPI, semantic Kuadrant cell; add `ClusterVersionsPanel.test.tsx` smoke asserting warning variant when `clusterReachable: false`
- [x] 3.4 Add i18n keys to `en.json` and `ja.json` (cluster connection banner, unreachable Kuadrant, operator absent, fallback default, profile assumed); verify `npm test` includes `locales.smoke.test.ts`

## 4. Verification and docs

- [x] 4.1 Run full backend suite `cd backend && mvn test` and frontend `cd frontend && npm test && npm run typecheck`
- [ ] 4.2 Manual check: `mvn quarkus:dev` without `oc login` → Connection shows unreachable banner, Compatibility shows cluster connection not Kuadrant missing; after `oc login` + refresh → Kuadrant version shown, compatibility clean
- [x] 4.3 Update `docs/api-spec.yml` cluster versions response schema if documented; link change to [#228](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/228) in PR description
