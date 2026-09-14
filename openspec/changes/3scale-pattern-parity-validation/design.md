# Design: 3scale Pattern Parity Validation (P1)

**Epic:** [#278](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/278)  
**Scope:** P1 only — #279, #280, #282, #281, #284 (`skip_specs: true`)

## Technical Approach

Catalog-driven **config parity** in PR CI: frozen `ApiService` export JSON per `rhcl_seed_*` → `ConversionService.convert` → fragment assertions + `ValidationService`. Convention-based paths; thin manifest for drift guard. Playwright migrates to shared contract in P2 (#283).

```
catalog.yaml ──► drift guard (sets match)
expectations.yaml ──┐
exports/*.json ─────┼──► SeedCatalogConversionIT (Failsafe *IT)
                    └──► SeedCatalogIntegrityTest (Surefire, no Quarkus)
```

## Architecture Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Fixture format | Jackson `ApiService` JSON (REST export shape) | Matches `GET /api/services/{id}`; no new DTO |
| Path convention | `testdata/exports/{system_name}.json` | Avoid per-row paths in manifest |
| Expectations | `testdata/seed/expectations.yaml` | Human-editable; migrate from `yaml-expectations.ts` |
| Manifest | `testdata/seed/manifest.yaml` — version + optional `cluster_profile` overrides only | Full index derived by convention; manifest not duplicated 21× |
| Assertion style | Fragment `mustContain` / `mustNotContain` | Existing E2E pattern; not full golden YAML |
| IT placement | `backend/src/test/e2e/.../SeedCatalogConversionIT.java` | Same as `MigrationWorkflowIT`; Failsafe `*IT` |
| Drift guard | `SeedCatalogIntegrityTest` (Surefire) | Fast; fails PR if catalog ≠ expectations keys ≠ export files |
| CI gate | Runs inside existing `mvn verify` (Failsafe) | No new job for P1; document in #284 / STATUS_CHECK_MATRIX |
| FE loader | `js-yaml` devDep; `e2e/load-expectations.ts` | No yaml dep today; P2 replaces inline `yaml-expectations.ts` |
| Refresh script | `scripts/refresh-seed-exports.sh` — manual, lab secrets | Lists services by `system_name`, saves JSON via running backend |

## Data Flow

```
refresh-seed-exports.sh (manual, THREESCALE_*)
  → GET /api/services?url=…  (match system_name)
  → GET /api/services/{id}
  → testdata/exports/{system_name}.json

SeedCatalogConversionIT (CI)
  → load JSON → ApiService (ObjectMapper)
  → conversionService.convert(svc, "seed-parity-ns")
  → validationService.validate(files)
  → assert fragments from expectations.yaml
```

## File Changes

| File | Action |
|------|--------|
| `testdata/seed/manifest.yaml` | Create — `version: 1`, optional profile overrides |
| `testdata/seed/expectations.yaml` | Create — 21 products; port from `yaml-expectations.ts` + stubs for rest |
| `testdata/exports/*.json` | Create — 21 fixtures (#280) |
| `scripts/refresh-seed-exports.sh` | Create — manual refresh; parses `products:` block via `scripts/lib/seed-catalog-products.sh` |
| `scripts/verify-seed-catalog-products.sh` | Create — shell drift guard (catalog parse ↔ expectations keys) |
| `backend/src/test/java/.../seed/SeedCatalogFixtureSupport.java` | Create — load JSON/YAML |
| `backend/src/test/java/.../seed/SeedCatalogIntegrityTest.java` | Create — drift guard (#279) |
| `backend/src/test/e2e/.../SeedCatalogConversionIT.java` | Create — parameterized parity (#281) |
| `frontend/e2e/load-expectations.ts` | Create (P2 slice ok in P1 if small) |
| `frontend/e2e/yaml-expectations.ts` | Modify — re-export from YAML (P2 #283) |
| `testdata/seed/README.md` | Modify — manifest/exports/expectations docs |
| `docs/STATUS_CHECK_MATRIX.md` | Modify — note SeedCatalog IT in backend verify (#284) |

## Interfaces / Contracts

```yaml
# testdata/seed/expectations.yaml (sketch)
version: 1
products:
  rhcl_seed_cors:
    productLabel: "RHCL Seed CORS"
    namespace: seed-parity-ns
    files:
      httproute.yaml:
        mustContain: ["kind: HTTPRoute"]
        mustNotContain: []
```

```java
// SeedCatalogFixtureSupport
static ApiService loadExport(String systemName);
static Map<String, ProductExpectation> loadExpectations();
static Set<String> catalogSystemNames(); // parse catalog.yaml
```

## Testing Strategy

| Layer | What | How |
|-------|------|-----|
| Surefire | Catalog ↔ expectations ↔ exports alignment | `SeedCatalogIntegrityTest` |
| Failsafe | Per-product convert + validate + fragments | `SeedCatalogConversionIT` @ParameterizedTest |
| Vitest (optional P1) | TS expectations loader smoke | Parse `expectations.yaml` keys |
| Playwright | Full wizard | P2 #283 / nightly #285 |

## Threat Matrix

N/A for CI test harness. **`refresh-seed-exports.sh`**: manual-only; requires `THREESCALE_ACCESS_TOKEN` in env (never commit); operator runs locally — not invoked in CI.

## Migration / Rollout

1. PR #289 (catalog chains) merge first.  
2. P1 apply: stacked PRs or single branch `feature/278-seed-pattern-parity-p1` closing #279–#282, #281, #284 in one or two PRs.  
3. Initial expectations for 5 E2E products fully populated; remaining 15 start with minimal `mustContain: [kind: HTTPRoute]` then tightened per policy PR.

## Open Questions

- [ ] **3scaleextract** chain policy defaults — seeder may need sibling-repo PR before live refresh of chain products (#289 catalog).
- [ ] Single PR vs two (harness+#279–#282 vs fixtures+#281) if diff >400 lines — default **single** unless review asks split.
