## 1. Baseline Measurement

- [x] 1.1 Create feature branch `feature/210-test-coverage-controller` from `main` and verify `mvn -Dtest='!PlaywrightE2EIT' verify` passes green on it
- [x] 1.2 Open JaCoCo HTML report at `backend/target/jacoco-merged-report/index.html`, record current line+branch coverage for the `controller` package, and identify uncovered lines per controller file

## 2. New Test: DefaultsController

- [x] 2.1 Create `DefaultsControllerTest.java` as `@QuarkusTest` with a test for GET `/api/defaults` when `threescale.default.url` and `threescale.default.token` are not configured — verify response is 200 with `threescale.configured == false` and `threescale.url == null` and `threescale.token == null`
- [x] 2.2 Add test for GET `/api/defaults` when config properties are set (via `application.properties` `%test` profile or `@TestProfile`) — verify response is 200 with `threescale.configured == true` and non-null url/token values
- [x] 2.3 Add test for GET `/api/defaults` when config properties are blank/whitespace strings — verify response filters them out (`configured == false`, url/token `null`)

## 3. Extend Existing Controller Tests (target uncovered branches)

- [x] 3.1 Review JaCoCo uncovered lines for `ApplyController` and add tests for uncovered error-handling branches (partial apply failures, individual resource apply exceptions) — verify the new methods increase branch coverage for `ApplyController`
- [x] 3.2 Review JaCoCo uncovered lines for `ConversionController` and add tests for uncovered branches (e.g., `dnsHostname` null path, bulk convert partial failure) — verify the new methods increase branch coverage for `ConversionController`
- [x] 3.3 Review JaCoCo uncovered lines for `ExportController` and add tests for uncovered branches (empty service list, pagination edge cases, CORS origins override paths) — verify the new methods increase branch coverage for `ExportController`
- [x] 3.4 Review JaCoCo uncovered lines for `ImportController` and add tests for uncovered branches (invalid file upload, parse errors, empty/malformed JSON) — verify the new methods increase branch coverage for `ImportController`
- [x] 3.5 Review JaCoCo uncovered lines for `ClusterController`, `ConnectionController`, `GatewayInfoController`, `HistoryController`, `PackageController`, `ValidationController` and add tests where coverage is below 90% — verify each controller reaches ≥90% line coverage
- [x] 3.6 Review JaCoCo uncovered lines for `SetupController` and add tests for remaining uncovered branches if any — verify `SetupController` reaches ≥90%

## 4. Verification

- [x] 4.1 Run `mvn -Dtest='!PlaywrightE2EIT' verify` and confirm all tests pass green with zero failures
- [ ] 4.2 Open JaCoCo HTML report and verify `controller` package line coverage is ≥90% and branch coverage is ≥85%
- [x] 4.3 Run Checkstyle verification and confirm zero violations in new/modified test files

### JaCoCo snapshot (post-apply, `feature/210-test-coverage-controller`)

| Metric | Value | Target |
|--------|-------|--------|
| Package lines | **95.3%** (783/822) | ≥90% ✅ |
| Package branches | **82.9%** (314/380) | ≥85% ❌ (gap: unreachable prefetch/validation branches) |

Per-controller line gaps still below 90%: `ApplyController` (88.8%), `ConversionController` (87.5%). Branch gap is concentrated in `ApplyController`, `ConversionController`, `SetupController`, and `ExportController` helper branches (`toPlural`, prefetch interrupt, etc.) that need many more `@QuarkusTest` scenarios or are unreachable via HTTP validation (`@NotBlank` on `serviceIds`).
