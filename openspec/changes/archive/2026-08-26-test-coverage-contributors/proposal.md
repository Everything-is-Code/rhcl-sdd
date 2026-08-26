# test-coverage-contributors

## Why

PR-5 of issue #210 (raise backend Codecov baseline). The `service/generator/contributor/` package contains 29 source files but only 3 have tests (AuthPolicyBuilder, HttpRouteBuilder, SecretBuilder). Contributors hold the core policy-mapping logic — untested regressions here silently corrupt generated YAML. Reaching ≥70% line coverage on this package unblocks the Codecov gate and protects future policy additions.

## What Changes

Add JUnit 5 unit tests for all untested contributor classes grouped by family (AuthPolicy, HTTPRoute, Secret, Other). No production code changes — test-only.

Depends on PR-3 (`test-coverage-conversion-support`) for shared test fixtures; can run in parallel with PR-4.

## Capabilities

### New Capabilities

None (`skip_specs: true`).

### Modified Capabilities

None.

## Impact

- **Coverage**: `service/generator/contributor/` line coverage raised from ~10% to ≥70%.
- **Risk**: Zero — test-only change, no production code modified.
- **Dependencies**: Requires PR-3 merged first (shared ConversionContext mock factory). Parallel-safe with PR-4.
