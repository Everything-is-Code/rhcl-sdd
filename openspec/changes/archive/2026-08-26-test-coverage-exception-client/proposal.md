# test-coverage-exception-client

## Why

Issue #210 aims to raise the backend Codecov baseline from ~37 % to tiered
per-package targets. This change (PR-2 of #210) focuses on two low-coverage
packages:

* **`exception/`** — 8 source files, only 2 have tests today (ErrorSanitizer,
  ApiExceptionMapperResource). The remaining 6 exception classes are untested.
* **`client/`** — 1 source file (ThreeScaleClient), 0 tests.

Without dedicated tests these packages drag down the overall baseline and
leave constructor / error-code logic and REST-client error handling unverified.

## What Changes

Test-only additions — no production code is modified.

* Unit tests for every `ApiException` subclass verifying constructors,
  `message`, and `errorCode()`.
* Extended branch coverage for `ErrorSanitizerTest` and
  `ApiExceptionMapperTest`.
* Integration test for `ThreeScaleClient` using WireMock to simulate the
  3scale REST API (happy paths + error codes 401, 404, 500).

Coverage targets: `exception/` ≥ 80 %, `client/` ≥ 60 %.

## Capabilities

### New Capabilities

None — test-only change.

### Modified Capabilities

None — test-only change.

## Impact

* **Risk**: minimal — no production code changes.
* **Coordination**: ThreeScaleClient tests should be aware of pagination
  improvements tracked in #169 to avoid conflicting test assumptions.
* **CI**: JaCoCo thresholds will be enforced after this PR merges.
