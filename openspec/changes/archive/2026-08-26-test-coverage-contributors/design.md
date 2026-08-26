# Design — test-coverage-contributors

## Approach

Plain JUnit 5 unit tests. Contributors implement two key methods — `shouldContribute(ConversionContext)` and `contribute(builder)` — with pure logic that maps 3scale policy state to Kuadrant resource fragments. Each test verifies both decision and output independently.

## Test Strategy

### Fixture Reuse

Leverage the shared test infrastructure from PR-3 (`test-coverage-conversion-support`):

- **ConversionContext mock factory** — builds minimal `ConversionContext` instances with configurable authentication type, enabled policies, and service metadata.
- **ApiService builder** — constructs `ApiService` with attached `Policy`, `Authentication`, `MappingRule`, and `Metric` objects for specific scenarios.
- **Builder spies** — real or lightweight `AuthPolicyBuilder`, `HttpRouteBuilder`, `SecretBuilder` instances to capture contributor output.

### Grouping by Family

| Family | Contributors | Count |
|--------|-------------|-------|
| AuthPolicy | AnonymousContributor, ApiKeyAuthenticationContributor, AppIdKeyAuthenticationContributor, AuthCachingContributor, EmptyAuthenticationContributor, JwtAuthenticationContributor, JwtClaimCheckContributor, KeycloakRoleCheckContributor, Oauth2IntrospectionContributor | 9 |
| HTTPRoute | CorsFiltersContributor, CorsOptionsContributor, HeaderModContributor, HttpRouteAnnotationsContributor, MappingRulesContributor, RetryContributor, TimeoutsContributor | 7 |
| Secret | AnonymousSecretContributor, ApiKeySecretContributor, AppIdKeySecretContributor, DefaultCredentialsSecretContributor, TokenIntrospectionSecretContributor | 5 |
| Other | IpCheckOpaContributor, AuthPolicyYamlMerger | 2 |

### Test Pattern per Contributor

Each test class follows the same structure:

1. **`shouldContribute_true`** — given a context where the contributor applies (correct auth type, policy enabled), assert `shouldContribute()` returns `true`.
2. **`shouldContribute_false`** — given a context where it does not apply, assert `false`.
3. **`contribute_addsExpectedFragments`** — call `contribute()` and assert the builder contains the expected YAML fragment / structure.
4. **Edge cases** — null policies, empty collections, conflicting settings.

### Parameterized Tests

Use `@ParameterizedTest` with `@MethodSource` for contributors that share patterns (e.g., secret contributors differ only in which credential fields they emit). This reduces boilerplate across the 5 secret contributors and the auth-type-gated AuthPolicy contributors.

## Constraints

- No production code changes.
- No CDI container — pure `new Contributor()` instantiation.
- No Quarkus `@QuarkusTest` overhead — fast unit tests only.
- Target: ≥70% line coverage on `service/generator/contributor/` package.
