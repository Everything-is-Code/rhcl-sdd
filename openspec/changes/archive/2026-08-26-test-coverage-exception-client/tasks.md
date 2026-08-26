# test-coverage-exception-client — Tasks



## 1. Baseline & Branch Setup



- [x] 1.1 Create feature branch `feature/210-test-coverage-exception-client` from `main` — verify branch exists and is clean.

- [x] 1.2 Run `mvn verify` + JaCoCo report — record current line-coverage percentages for `exception/` and `client/` packages as the baseline.

  **Baseline (`main`, clean exec):** `exception/` 89 % lines (75/84), 60 % branches; `client/` n/a (interface, no tests). See `design.md` § JaCoCo baseline.



## 2. Exception Subclass Unit Tests



- [x] 2.1 Create `ApiExceptionTest` — verify constructor sets message and `errorCode()` returns expected value.

- [x] 2.2 Create `ClusterApplyExceptionTest` — verify constructors (message, message+cause), `getMessage()`, `errorCode()`.

- [x] 2.3 Create `ImportParseExceptionTest` — verify constructors, `getMessage()`, `errorCode()`.

- [x] 2.4 Create `NotFoundExceptionTest` — verify constructors, `getMessage()`, `errorCode()`.

- [x] 2.5 Create `ThreeScaleClientExceptionTest` — verify constructors (message, message+cause), `getMessage()`, `errorCode()`.

- [x] 2.6 Create `ValidationExceptionTest` — verify constructors, `getMessage()`, `errorCode()`, and any validation-items accessors.



## 3. Extend Existing Exception Tests



- [x] 3.1 Extend `ErrorSanitizerTest` — add cases for uncovered branches (null input, edge-case patterns) — verify by checking JaCoCo branch coverage improves.

- [x] 3.2 Extend `ApiExceptionMapperTest` — add cases for exception types not yet exercised through `ExceptionTestResource` — verify all mapper response codes are tested.



## 4. ThreeScaleClient Integration Test



- [x] 4.1 Create `ThreeScaleClientTest` with `@QuarkusTest` + WireMock — stub `getServices` and `getServiceById` happy paths, assert correct deserialization.

- [x] 4.2 Add WireMock error stubs — test HTTP 401 (unauthorized), 404 (not found), 500 (server error) responses produce expected exceptions.

- [x] 4.3 Add at least one additional method test (e.g. `getApplicationPlans` or `getBackendUsages`) — verify by assertion on returned data.



## 5. Verification



- [x] 5.1 Run `mvn verify` — all tests green, zero failures.

- [x] 5.2 Run JaCoCo report — confirm `exception/` ≥ 80 % line coverage, `client/` ≥ 60 % line coverage.

- [x] 5.3 Run Checkstyle — zero new violations in test files.

