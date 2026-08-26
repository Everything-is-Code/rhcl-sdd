## 1. Baseline Measurement

- [x] 1.1 Create feature branch `feature/210-test-coverage-conversion-support` from `main` and verify `mvn -Dtest='!PlaywrightE2EIT' verify` passes green on it
- [x] 1.2 Open JaCoCo HTML report at `backend/target/jacoco-merged-report/index.html`, record current line+branch coverage for the `service/conversion/` package, and identify uncovered lines per source file

## 2. High Priority Tests (6 classes)

- [x] 2.1 Create `BackendResolverTest.java` — test backend resolution logic for different backend configurations (single backend, multiple backends, null/missing backends, backend type detection) — verify the new tests increase line coverage for `BackendResolver`
- [x] 2.2 Create `ConversionContextTest.java` — test context construction, option accessors, service/backend retrieval, edge cases (empty collections, null options, missing fields) — verify the new tests increase line coverage for `ConversionContext`
- [x] 2.3 Create `PolicyConfigSupportTest.java` — test policy configuration extraction, policy chain parsing, empty/null policy lists, unknown policy types — verify the new tests increase line coverage for `PolicyConfigSupport`
- [x] 2.4 Create `HttpRouteSupportTest.java` — test HTTP route path building, mapping rule translation, path prefix/exact matching, wildcard handling — verify the new tests increase line coverage for `HttpRouteSupport`
- [x] 2.5 Create `AuthPolicySupportTest.java` — test auth policy generation for different authentication types (API key, OIDC/JWT, app ID + app key), missing credentials, combined auth scenarios — verify the new tests increase line coverage for `AuthPolicySupport`
- [x] 2.6 Create `SecretSupportTest.java` — test secret generation for different credential types, empty/null values, secret naming conventions — verify the new tests increase line coverage for `SecretSupport`

## 3. Medium Priority Tests (4 classes)

- [x] 3.1 Create `ConversionYamlSupportTest.java` — test YAML serialization helpers, null handling, empty map/list serialization, multiline string formatting — verify the new tests increase line coverage for `ConversionYamlSupport`
- [x] 3.2 Create `ReadmeSupportTest.java` and/or `ReadmeNotesTest.java` — test readme section generation, note accumulation, note deduplication, empty notes handling — verify the new tests increase line coverage for `ReadmeSupport` and `ReadmeNotes`
- [x] 3.3 Create `JwtClaimCheckSupportTest.java` — test JWT claim parsing, claim pattern matching, malformed claim strings, nested claim paths — verify the new tests increase line coverage for `JwtClaimCheckSupport`
- [x] 3.4 Create `RateLimitSupportTest.java` — test rate limit computation from plan limits, period normalization, missing/zero limits, multiple plan ceiling scenarios — verify the new tests increase line coverage for `RateLimitSupport`

## 4. Low Priority Tests (4 classes)

- [x] 4.1 Create `ContributorOrderingTest.java` — test ordering comparator logic, equal-priority contributors, null-safe comparisons — verify the new tests increase line coverage for `ContributorOrdering`
- [x] 4.2 Create `BackendTypeTest.java` — test enum values, `fromString` or factory methods if any, coverage of all enum constants — verify the new tests increase line coverage for `BackendType`
- [x] 4.3 Create `RegistryDiscoveryMarkersTest.java` — test marker interface/data accessors, default values, builder if present — verify the new tests increase line coverage for `RegistryDiscoveryMarkers`
- [x] 4.4 Create `ResolvedBackendTest.java` — test data holder construction, accessors, equality, null fields — verify the new tests increase line coverage for `ResolvedBackend`

## 5. Verification

- [x] 5.1 Run `mvn -Dtest='!PlaywrightE2EIT' verify` and confirm all tests pass green with zero failures
- [x] 5.2 Open JaCoCo HTML report and verify `service/conversion/` package line coverage is ≥50%
- [x] 5.3 Run Checkstyle verification and confirm zero violations in new test files
