# Proposal: 3scale Pattern Parity Validation

**Epic:** [#278](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/278)  
**SDD change:** `3scale-pattern-parity-validation`  
**Exploration:** `exploration.md` (2026-09-03)

## Intent

Wire the existing `rhcl_seed_*` lab catalog into an **automated parity matrix**: each 3scale seed product must produce the expected RHCL YAML configuration (Phase 1), then optionally prove runtime behavior on a cluster (Phase 3).

Catalog prep: three **multi-policy chain** products added to `testdata/seed/catalog.yaml` (`claim_role_chain`, `claim_cache_chain`, `auth_chain`) to align with Playwright E2E expectations.

## GitHub backlog

| Phase | Issue | Title |
|-------|-------|-------|
| Epic | **#278** | 3scale seed catalog → RHCL pattern parity matrix |
| P1 | **#279** | seed catalog manifest + drift guard |
| P1 | **#280** | frozen Admin API export JSON per product |
| P1 | **#281** | `SeedCatalogConversionIT` parameterized |
| P1 | **#282** | shared YAML expectation contract (BE + FE) |
| P2 | **#283** | align Playwright with full catalog |
| P1 | **#284** | PR CI job `seed-catalog-parity` |
| P2 | **#285** | nightly Playwright E2E (#202) |
| P3 | **#286** | runtime parity spike (routing) |
| P3 | **#287** | runtime parity (auth class) |
| P3 | **#288** | cluster profile matrix |

## Locked P1 scope (first apply slice)

Implement **#279 → #282 → #281 → #284** in order (manifest + exports + shared contract + IT + CI).  
**#283** and **#285** follow in P2. **#286–#288** deferred.

## In scope (P1)

- `testdata/seed/manifest.yaml` + drift guard tests
- `testdata/exports/*.json` + `scripts/refresh-seed-exports.sh`
- `testdata/seed/expectations.yaml` (shared contract)
- `SeedCatalogConversionIT` + PR CI job
- Docs: `testdata/seed/README.md`, `STATUS_CHECK_MATRIX.md` row when stable

## Out of scope (P1)

- Runtime HTTP parity (#286, #287)
- Playwright full catalog (#283) — P2 unless bundled after contract lands
- `3scaleextract` policy default changes for chain products (follow-up PR in sibling repo if seeder needs new `policyConfigurations`)
- Golden full-file YAML snapshots (use fragment contracts)

## Success criteria

- Every `system_name` in `catalog.yaml` (21 products) has export fixture, expectation entry, and green `SeedCatalogConversionIT`
- PR CI fails on catalog/expectation drift or conversion regression
- Epic #278 P1 child issues closable via PR(s) referencing change name

## Delivery

- Branch: `feature/278-3scale-pattern-parity-validation` (or stacked PRs per issue if >400 lines)
- PR body: `Part of #278`, `Closes #279` (etc. per slice)
- Reviewer: @pcastelo

## Ready for design

**Yes** — proceed `/opsx-design` with P1 locked; P2/P3 documented only.
