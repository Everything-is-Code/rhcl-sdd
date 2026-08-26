# Tasks — test-coverage-service-data-layers

GitHub: [#222](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/222)

## Prerequisites

- [ ] #210 tiers merged on `main` (controller, exception/client, conversion-support, generators, contributors)
- [ ] `design.md` reviewed
- [ ] Product branch before first `/opsx-apply` commit: `feature/222-test-coverage-data-layers` (slice 1; other slices use `feature/222-*` per design)

## 1. Baseline

- [ ] `git checkout main && git pull`
- [ ] `cd backend && mvn -Dtest='!PlaywrightE2EIT' verify`
- [ ] Record JaCoCo line % for `service`, `dto`, `model`, `entity` in verify-report or issue comment

## 2. Slice 1 — dto / model / entity (PR-1)

- [ ] If `dto` ≥95% and `entity` ≥70% — document skip, close slice with comment only
- [ ] Else extend `ModelTest` / `BeanValidationDtoTest` for uncovered paths
- [ ] Else add `entity/*EntityTest` with `@QuarkusTest` + H2
- [ ] PR: `Closes part of #222`

## 3. Slice 2 — ThreeScaleExportService (PR-2) **required for epic**

- [ ] `ThreeScaleExportServiceTest` or `@QuarkusTest` + WireMock covering main export paths
- [ ] Error paths: 401/404/500 where applicable
- [ ] Target: `service` package lines ≥40% (or document partial + follow-up)
- [ ] PR: `Closes part of #222`

## 4. Slice 3 — ClusterVersionService + CompatibilityService (PR-3)

- [ ] Unit / `@QuarkusTest` with mocked K8s client
- [ ] Target: combined ≥50% lines for those classes
- [ ] PR: `Closes part of #222`

## 5. Slice 4 — ConversionService integration (PR-4, optional)

- [ ] Extend `ConversionServiceTest` for orchestration paths not hit by contributor unit tests
- [ ] PR: `Closes part of #222`

## 6. Verification

- [ ] `mvn -Dtest='!PlaywrightE2EIT' verify` green
- [ ] Codecov project `backend` higher than pre-#222 baseline
- [ ] `/verify` + verify-report on this change
- [ ] Comment on #222 with before/after table
- [ ] `/opsx-archive` this change

## Docs

- [ ] Update `docs/sdd-backlog.md` when slices merge
