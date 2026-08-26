## Why

Codecov baseline on `main` is ~37.2% for the backend Java tree. The `controller` package sits at ~83% (JaCoCo) but the Codecov ratchet must climb without blocking unrelated PRs. Raising `controller` to ≥90% is the highest-ROI first step because the REST surface has the most direct user impact and the gap is small. This is **Tier A** of issue [#210](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/210).

## What Changes

- **New test**: `DefaultsControllerTest.java` — the only controller without any test coverage (GET `/api/defaults` with optional 3scale URL/token config properties).
- **Extended tests**: Add edge-case test methods to existing controller tests to cover uncovered branches (error handling, null/empty inputs, partial result edge cases, boundary conditions).
- **No production code changes** — test-only, unless a minimal tweak is required for testability (e.g., package-private visibility).

## Capabilities

### New Capabilities

_None — this change adds tests only, no new behavioral capabilities._

### Modified Capabilities

_None — no spec-level behavior changes. `skip_specs: true` is set in `.openspec.yaml` because this is a pure test/quality change._

## Impact

- **Code**: `backend/src/test/java/com/redhat/migrationtoolkit/rhcl/controller/` — new + extended test files
- **CI**: Codecov `codecov/project/backend` baseline rises after merge; no gate changes
- **Dependencies**: None — tests use existing `@QuarkusTest` + REST-assured infrastructure
- **APIs**: No changes
- **Related issues**: Part of [#210](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/210) (Tier A); does not affect #169, #149, or #198
