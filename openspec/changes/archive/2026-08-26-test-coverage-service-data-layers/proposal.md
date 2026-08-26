# test-coverage-service-data-layers

## Why

[#210](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/210) raised tiered coverage (controller, conversion, generators, contributors) from **~37%** to **~50–60%** Codecov project `backend`. The remaining gap is dominated by **`com.redhat.migrationtoolkit.rhcl.service`** root (~19% lines, ~1288 LOC in JaCoCo) — especially `ThreeScaleExportService` (~1000 LOC).

Optional PR-6 `test-coverage-dto-model-entity` is **superseded here**: `dto` is already 100% lines; `model`/`entity` are small packages bundled in PR slice 1.

## What Changes

Test-only work in up to **four PR slices** (see [#222](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/222)):

1. **Data layers** — extend `DtoTest` / `ModelTest` / `BeanValidationDtoTest`; add entity `@QuarkusTest` only if JaCoCo gaps remain after baseline.
2. **`ThreeScaleExportService`** — WireMock integration tests (coordinate with #169 bulk-fetch patterns).
3. **`ClusterVersionService` + `CompatibilityService`** — mocked K8s / fixture data paths.
4. **`ConversionService`** — integration scenarios beyond contributor unit tests (best effort).

Minimal production seams only when required for deterministic tests (pattern: `GatewayDnsResolver` in #213).

## Capabilities

None — `skip_specs: true`.

## Impact

- **Code**: `backend/src/test/java/**` only (unless tiny testability seam).
- **CI**: Codecov `backend` project ratchet; no threshold edits.
- **Supersedes**: OpenSpec `test-coverage-dto-model-entity` (archived, not implemented).
- **Related**: #169 (export performance — avoid duplicate WireMock pagination work), #198 (JaCoCo hard floor).
