# Design — test-coverage-cluster-compat

## Context

`ClusterVersionServiceTest` (~800 LOC) and `CompatibilityServiceTest` (~800 LOC) already exist from #210 era. `ConversionServiceTest` is large; gaps are orchestration overloads and edge branches.

## PR strategy

| Slice | Branch | Focus |
|-------|--------|-------|
| 3 | `feature/222-test-coverage-cluster-compat` | `ClusterVersionService` + `CompatibilityService` unit tests |
| 4 | same branch or `feature/222-test-coverage-conversion-service` | `ConversionService` overload/orchestration |

Single PR acceptable if diff stays reviewable (&lt;300 lines).

## Key decisions

1. **Extend existing test classes** — do not duplicate Mockito K8s setup.
2. **Assert behavior** — status strings, capability tags, YAML fragments (not only non-null).
3. **Skip if JaCoCo already ≥50%** for target classes on branch baseline.

## Test location

`backend/src/test/java/com/redhat/migrationtoolkit/rhcl/service/`
