# Tasks — test-coverage-contributors

## 1. Baseline setup

- [x] Create feature branch from PR-3 branch (requires `test-coverage-conversion-support` merged)
- [x] Confirm JaCoCo baseline for `service/generator/contributor/` package
- [x] Verify shared test fixtures from PR-3 are available

## 2. Test fixtures (contributor-specific)

- [x] Create helper methods for building `ConversionContext` with specific authentication types (apiKey, appIdKey, jwt, oauth2, keycloak, anonymous)
- [x] Create helper builders for `Policy` objects with specific configuration maps
- [x] Create lightweight builder/spy wrappers if needed for asserting contributor output

## 3. AuthPolicy contributor tests (9 classes)

- [x] `AnonymousContributorTest`
- [x] `ApiKeyAuthenticationContributorTest`
- [x] `AppIdKeyAuthenticationContributorTest`
- [x] `AuthCachingContributorTest`
- [x] `EmptyAuthenticationContributorTest`
- [x] `JwtAuthenticationContributorTest`
- [x] `JwtClaimCheckContributorTest`
- [x] `KeycloakRoleCheckContributorTest`
- [x] `Oauth2IntrospectionContributorTest`

## 4. HTTPRoute contributor tests (7 classes)

- [x] `CorsFiltersContributorTest`
- [x] `CorsOptionsContributorTest`
- [x] `HeaderModContributorTest`
- [x] `HttpRouteAnnotationsContributorTest`
- [x] `MappingRulesContributorTest`
- [x] `RetryContributorTest`
- [x] `TimeoutsContributorTest`

## 5. Secret contributor tests (5 classes)

- [x] `AnonymousSecretContributorTest`
- [x] `ApiKeySecretContributorTest`
- [x] `AppIdKeySecretContributorTest`
- [x] `DefaultCredentialsSecretContributorTest`
- [x] `TokenIntrospectionSecretContributorTest`

## 6. Other contributor tests (2 classes)

- [x] `IpCheckOpaContributorTest`
- [x] `AuthPolicyYamlMergerTest`

## 7. Verification

- [x] `mvn verify` passes (all tests green)
- [x] JaCoCo report shows `service/generator/contributor/` ≥70% line coverage
- [x] Checkstyle clean (no new violations)
- [x] No production code changes in diff
