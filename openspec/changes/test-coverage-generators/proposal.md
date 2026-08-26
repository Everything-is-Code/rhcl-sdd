## Why

The `service/generator/` package contains 25 source files responsible for producing every Kubernetes YAML output during 3scale → Kuadrant conversion — yet only 1 of those files (`ResourceGeneratorRegistry`) has a dedicated test. Existing indirect coverage (via `ResourceGeneratorRegistryTest`, `*DiscoveryTest`, `IsolatedGeneratorTest`, `ManualCdiParityTest`) exercises CDI wiring and registry lookup but does not verify the YAML output correctness of individual generators. This is **PR-4** of issue [#210](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/210) (raise backend Codecov baseline).

## What Changes

- **New tests**: One dedicated `@QuarkusTest` per generator class in `service/generator/` (19 generators total, excluding 4 factory classes and 1 interface).
- **No production code changes** — test-only. `skip_specs: true`.
- **Depends on** PR-3 (`test-coverage-conversion-support`) since generators delegate to `service/conversion/*` helpers that PR-3 tests and potentially stabilizes.

## Capabilities

### New Capabilities

_None — this change adds tests only, no new behavioral capabilities._

### Modified Capabilities

_None — no spec-level behavior changes. `skip_specs: true` is set in `.openspec.yaml` because this is a pure test/quality change._

## Impact

- **Code**: `backend/src/test/java/com/redhat/migrationtoolkit/rhcl/service/generator/` — 19 new test files
- **CI**: Codecov `codecov/project/backend` baseline rises after merge; `service/generator/` package target ≥70%
- **Dependencies**: Uses existing `@QuarkusTest` + CDI injection; mocks `ConversionContext` with controlled `ApiService`, `Backend`, `ConversionOptions`
- **APIs**: No changes
- **Exclusions**: Factory classes (`Manual*Factory`) and the `ResourceGenerator` interface are excluded — factories are trivial CDI producers, interface has no logic
- **Related issues**: Part of [#210](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/210) (PR-4); depends on PR-3 (`test-coverage-conversion-support`); does not affect #169, #149, or #40
