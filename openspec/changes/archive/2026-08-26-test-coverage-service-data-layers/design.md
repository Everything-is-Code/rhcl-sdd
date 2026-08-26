# Design — test-coverage-service-data-layers

## Baseline (`main` after #210, Aug 2026)

JaCoCo merged report (`mvn -Dtest='!PlaywrightE2EIT' verify`):

| Package | Lines (approx.) | Notes |
|---------|-----------------|-------|
| `service` (root) | **19%** | Primary ROI — `ThreeScaleExportService`, `ClusterVersionService`, `CompatibilityService` |
| `model` | **50%** | Extend `ModelTest` |
| `entity` | **65%** | Optional Panache round-trip tests |
| `dto` | **100%** | Likely skip PR-1 implementation |

Global lines ~**60%** (2507/4135); Codecov project ~50–60%.

## PR strategy

One PR per slice; merge sequentially so Codecov ratchets.

| Slice | Branch hint | Test approach |
|-------|---------------|---------------|
| 1 | `feature/222-test-coverage-data-layers` | Extend aggregate tests; `@QuarkusTest` + H2 for entities if &lt;70% |
| 2 | `feature/222-test-coverage-export-service` | `@QuarkusTest` + WireMock for 3scale Admin API; reuse `WireMockThreeScaleResource` patterns from #214 |
| 3 | `feature/222-test-coverage-cluster-compat` | `@InjectMock KubernetesClient` / route fixtures |
| 4 | `feature/222-test-coverage-conversion-service` | `ConversionServiceTest` scenarios |

## Key decisions

1. **Measure before writing** — each slice starts with JaCoCo package % on feature branch; skip slice if target already met.
2. **No mega-PR** — `ThreeScaleExportService` alone warrants its own PR.
3. **#169 coordination** — do not duplicate pagination refactors; tests should survive #169 export changes or land after it.
4. **Assert content, not only kind/name** — follow review norm from #217/#218 (YAML fields, error codes, status codes).

## Test location

`backend/src/test/java/com/redhat/migrationtoolkit/rhcl/` — follow `testing-standards.mdc` naming (`*Test.java`, `method_condition_result`).
