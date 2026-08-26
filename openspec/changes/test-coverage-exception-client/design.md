# test-coverage-exception-client — Design

## Context

The `exception/` package contains 8 source files:

| Class | Baseline (pre-change) | After this change |
|-------|----------------------|-------------------|
| ApiException | No dedicated unit test | `ApiExceptionTest` |
| ApiExceptionMapperResource | `ApiExceptionMapperTest` + `ExceptionTestResource` (partial) | Extended mapper cases |
| ClusterApplyException | No dedicated unit test | `ClusterApplyExceptionTest` |
| ErrorSanitizer | `ErrorSanitizerTest` (partial branches) | Extended branch cases |
| ImportParseException | No dedicated unit test | `ImportParseExceptionTest` |
| NotFoundException | No dedicated unit test | `NotFoundExceptionTest` |
| ThreeScaleClientException | No dedicated unit test | `ThreeScaleClientExceptionTest` |
| ValidationException | No dedicated unit test | `ValidationExceptionTest` |

The `client/` package contains 1 source file:

| Class | Baseline (pre-change) | After this change |
|-------|----------------------|-------------------|
| ThreeScaleClient | No test (`@RegisterRestClient` interface — no executable JaCoCo lines) | `ThreeScaleClientTest` + `WireMockThreeScaleResource` |

Existing test pattern in the project: `@QuarkusTest` + REST-assured for
integration tests. `ApiExceptionMapperTest` already uses this pattern with an
`ExceptionTestResource` JAX-RS helper that throws exceptions to exercise the
mapper.

## JaCoCo baseline (pre-change, `main`)

Measured on `main` with a clean `jacoco-merged.exec` (`mvn -Dtest='!PlaywrightE2EIT' verify`).

| Package | Line coverage | Branch coverage | Notes |
|---------|---------------|-----------------|-------|
| `exception/` | **89 %** (75 / 84 lines) | **60 %** (6 / 10) | Mapper + sanitizer partially covered via integration tests; no subclass unit tests; `ClusterApplyException` at 48 % |
| `client/` | **n/a** (0 executable lines) | n/a | `ThreeScaleClient` is an interface — JaCoCo does not report the package; no WireMock / REST client test |

Gap summary before this change:

* **Exception subclasses** — exercised only indirectly through `ApiExceptionMapperTest`; constructors and `getCode()` / `getDetails()` not unit-tested per class.
* **ErrorSanitizer** — good line coverage (~91 %) but branch gaps (`sanitizeExceptionMessage(null)`, some token patterns).
* **ApiExceptionMapper** — missing paths: `ClusterApplyException` with details, custom error codes, sanitized messages in envelope.
* **ThreeScaleClient** — zero dedicated tests; client usage only mocked in `ThreeScaleExportServiceTest`.

## JaCoCo after implementation (feature branch)

| Package | Line coverage | Branch coverage | Notes |
|---------|---------------|-----------------|-------|
| `exception/` | **91 %** (77 / 84 lines) | **60 %** (6 / 10) | Meets ≥ 80 % goal |
| `client/` | n/a (interface) | n/a | API exercised by `ThreeScaleClientTest` (WireMock): `getServices`, `getService`, `getApplicationPlans`, `getBackendUsages`, HTTP 401/404/500 |

## Goals / Non-Goals

**Goals**

* `exception/` package line coverage ≥ 80 %.
* `client/` package line coverage ≥ 60 % (via integration tests against the REST client API; JaCoCo may show n/a for the interface).
* All new tests pass in CI (`mvn verify` green, Checkstyle clean).

**Non-Goals**

* Modifying any production source code.
* Reaching 100 % coverage — diminishing returns on simple getters.
* Testing ThreeScaleClient pagination edge cases (deferred to #169).

## Decisions

1. **Exception subclass tests are plain unit tests.** Each `ApiException`
   subclass is a simple POJO — a JUnit 5 test verifying constructors,
   `getMessage()`, and `getCode()` is sufficient. No need for `@QuarkusTest`
   overhead.

2. **Extend existing tests for uncovered branches.** `ErrorSanitizerTest` and
   `ApiExceptionMapperTest` already exist; add cases for branches not yet
   covered rather than creating parallel test classes.

3. **ThreeScaleClient uses WireMock inside `@QuarkusTest`.** The client is a
   CDI-managed REST client; WireMock stubs simulate the 3scale Admin API so
   tests run without a real 3scale instance. Key methods to cover: at minimum
   `getServices`, `getService`, plus error-handling paths (HTTP 401, 404,
   500).

4. **One test class per exception subclass.** Keeps test failures isolated and
   names self-documenting (e.g. `ClusterApplyExceptionTest`).

## Risks / Trade-offs

* **WireMock port conflicts** — use `@WireMockTest` or dynamic port binding to
  avoid clashes with Quarkus' test port.
* **ThreeScaleClient internals may change with #169** (pagination
  improvements). Write tests against the public API surface so they remain
  valid after #169.
* **Minimal risk overall** — test-only change with no production code edits.
