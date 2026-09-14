# Tasks: 3scale Pattern Parity Validation (P1)

## Prerequisites

- Store change: `3scale-pattern-parity-validation`; `skip_specs: true`
- Epic: [#278](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/278)
- Depends on: [#289](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/289) catalog chain products merged
- Product branch: `feature/278-seed-pattern-parity-p1` from latest `main`

## Review Workload Forecast

| Field | Value |
|-------|-------|
| Estimated changed lines | ~800–1200 (21 JSON fixtures + harness + expectations) |
| 400-line budget risk | **High** |
| Chained PRs recommended | **Yes** — PR-A harness + expectations; PR-B fixtures + IT |
| Delivery strategy | chained-to-main |
| Chain strategy | stacked-to-main |

Decision needed before apply: No (split if diff >400 at apply time)  
Ready for Apply: **Yes**

### Suggested Work Units

| Unit | Goal | PR | Closes | Test command |
|------|------|-----|--------|--------------|
| A | Manifest, expectations YAML, drift guard, fixture support | PR-A (#290) | Part of #279, #282 | `mvn -Dtest=SeedCatalogIntegrityTest test` |
| B | Export JSON fixtures + refresh script | PR-B | #280 | manual `refresh-seed-exports.sh` + integrity test |
| C | SeedCatalogConversionIT + matrix docs | PR-C | #281, #284 | `mvn -Dtest='!PlaywrightE2EIT' verify` |

## Phase 1: Harness (#279, #282)

- [x] 1.1 Branch `feature/278-seed-pattern-parity-p1` from `main` (after #289)
- [x] 1.2 Add `testdata/seed/manifest.yaml` (version + optional profile overrides)
- [x] 1.3 Add `testdata/seed/expectations.yaml` — port 5 E2E products; stub remaining 15 with minimal fragments
- [x] 1.4 Add `SeedCatalogFixtureSupport` + YAML/catalog parsers in `backend/src/test/java/.../seed/`
- [x] 1.5 RED→GREEN: `SeedCatalogIntegrityTest` — catalog keys = expectation keys (= export filenames when #280 lands)
- [x] 1.6 Update `testdata/seed/README.md` (manifest, expectations, conventions)
- [x] 1.7 PR #290 review remediation — fix `refresh-seed-exports.sh` catalog parse (`products:` block only); add `scripts/lib/seed-catalog-products.sh` + `verify-seed-catalog-products.sh`; PR body `Part of #279/#282` (not full close); 21-product count

## Phase 2: Frozen exports (#280)

- [x] 2.1 Add `scripts/refresh-seed-exports.sh` (manual; `THREESCALE_*` + backend URL)
- [ ] 2.2 Generate `testdata/exports/{system_name}.json` for all 21 products (lab or hand-crafted from export API)
- [ ] 2.3 Integrity test green with all export files present
- [ ] 2.4 Document refresh procedure in README

## Phase 3: Conversion IT (#281)

- [ ] 3.1 RED: `SeedCatalogConversionIT` — one failing product proves harness
- [ ] 3.2 GREEN: all 21 products — convert + `ValidationService` + fragment asserts
- [ ] 3.3 Tighten stub expectations where minimal fragments insufficient
- [ ] 3.4 `cd backend && mvn -Dtest='!PlaywrightE2EIT' verify`

## Phase 4: CI docs (#284)

- [ ] 4.1 Confirm `SeedCatalogConversionIT` runs under Failsafe in `backend-tests-cov`
- [ ] 4.2 Add row/note to `docs/STATUS_CHECK_MATRIX.md`
- [ ] 4.3 Open PR(s): `Part of #278`, `Part of #279` / `Part of #282` for PR-A harness; close issues when AC complete; CI green

## Phase 5: Deferred (P2/P3 — not this apply)

- [ ] 5.1 #283 — Playwright loads `expectations.yaml`; full 21-product E2E
- [ ] 5.2 #285 — nightly workflow
- [ ] 5.3 #286–#288 — runtime parity + profile matrix
