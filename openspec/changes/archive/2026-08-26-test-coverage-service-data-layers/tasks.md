# Tasks — test-coverage-service-data-layers

GitHub: [#222](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/222)

## Prerequisites

- [x] #210 tiers merged on `main` (controller, exception/client, conversion-support, generators, contributors)
- [x] `design.md` reviewed
- [x] Product branch before first `/opsx-apply` commit: `feature/222-test-coverage-data-layers` (slice 1; other slices use `feature/222-*` per design)

## 1. Baseline

- [x] `git checkout main && git pull`
- [x] `cd backend && mvn -Dtest='!PlaywrightE2EIT' verify`
- [x] Record JaCoCo line % for `service`, `dto`, `model`, `entity` in verify-report or issue comment

## 2. Slice 1 — dto / model / entity (PR-1)

- [x] If `dto` ≥95% and `entity` ≥70% — document skip, close slice with comment only
- [x] Else extend `ModelTest` / `BeanValidationDtoTest` for uncovered paths
- [x] Else add `entity/*EntityTest` with `@QuarkusTest` + H2
- [x] PR: `Closes part of #222` — [#225](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/225)

## 3. Slice 2 — ThreeScaleExportService (PR-2) **required for epic**

- [x] `ThreeScaleExportServiceTest` or `@QuarkusTest` + WireMock covering main export paths
- [x] Error paths: 401/404/500 where applicable
- [x] Target: `service` package lines ≥40% (or document partial + follow-up) — partial; JaCoCo post-merge pending
- [x] PR: `Closes part of #222` — [#224](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/224)

## 4. Slice 3 — ClusterVersionService + CompatibilityService (PR-3)

- [ ] Unit / `@QuarkusTest` with mocked K8s client — **deferred** (remain on #222)
- [ ] Target: combined ≥50% lines for those classes
- [ ] PR: `Closes part of #222`

## 5. Slice 4 — ConversionService integration (PR-4, optional)

- [ ] Extend `ConversionServiceTest` for orchestration paths not hit by contributor unit tests — **deferred**
- [ ] PR: `Closes part of #222`

## 6. Verification

- [x] `mvn -Dtest='!PlaywrightE2EIT' verify` green (per-slice in worktrees)
- [ ] Codecov project `backend` higher than pre-#222 baseline — pending PR merge
- [ ] `/verify` + verify-report on this change — partial (slices 1–2 only)
- [x] Comment on #222 with before/after table
- [x] `/opsx-archive` this change

## Docs

- [x] Update `docs/sdd-backlog.md` when slices merge
