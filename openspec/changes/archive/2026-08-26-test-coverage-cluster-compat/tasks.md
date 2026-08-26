# Tasks — test-coverage-cluster-compat

GitHub: [#222](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/222)

## Prerequisites

- [x] Slices 1–2 merged or PRs open (#224, #225)
- [x] `design.md` reviewed
- [x] Product branch: `feature/222-test-coverage-cluster-compat`

## 1. Slice 3 — ClusterVersionService + CompatibilityService

- [x] Extend `ClusterVersionServiceTest` — `ossmCompatibilityTable`, `sanitize`, `compareVersions`, `capabilitiesFrom` retries
- [x] Extend `CompatibilityServiceTest` — logging `json_object_config` WARNING, MEDIUM score boundary
- [x] Target: combined ≥50% lines for those classes (or document baseline if already met)
- [ ] PR: `Closes part of #222` — [#226](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/226)

## 2. Slice 4 — ConversionService orchestration

- [x] Extend `ConversionServiceTest` — string overloads (`loggingTarget`, `anonymousTarget`), `includeMigratedFromLabel`
- [ ] PR: `Closes part of #222` — [#226](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/226)

## 3. Verification

- [x] `mvn -Dtest='!PlaywrightE2EIT' verify` green
- [ ] Comment on #222 with JaCoCo delta if measurable
- [x] `/opsx-archive` this change

## Docs

- [ ] Update `docs/sdd-backlog.md` on merge
