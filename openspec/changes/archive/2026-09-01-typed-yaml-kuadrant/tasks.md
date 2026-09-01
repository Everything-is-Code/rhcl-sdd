# Tasks — typed-yaml-kuadrant

GitHub: [#262](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/262) — Phase 3

**Size note**: largest phase (~18 classes). Consider splitting into two PRs (sections 1-2 vs 3-4) if review load is too high.

## Prerequisites

- [x] 0.1 Confirm `typed-yaml-infra` (phase 0) is merged — `model/kuadrant/` records and `ManifestSerializer` available
- [x] 0.2 Create feature branch: `feature/262-typed-yaml-kuadrant` from current `main`
- [x] 0.3 Confirm CI green on `main`

## 1. AuthPolicy — builder and merger

- [x] 1.1 Rewrite `AuthPolicyBuilder.java` with typed accumulator methods (`addAuthentication(name, AuthenticationRule)`, `addAuthorization(name, AuthorizationRule)`, `setResponse(...)`) backed by `LinkedHashMap` — `build()` produces `AuthPolicyManifest` record — verify `AuthPolicyBuilderTest` passes
- [x] 1.2 Rewrite `AuthPolicyYamlMerger.java` to merge typed named-rule maps instead of string searching — verify `AuthPolicyYamlMergerTest` passes
- [x] 1.3 Rewrite `AuthPolicyGenerator.java` to serialize via `ManifestSerializer.toYaml(manifest)` — verify `AuthPolicyGeneratorTest` passes
- [x] 1.4 Confirm `AuthPolicyContributorDiscoveryTest` still passes unmodified

## 2. AuthPolicy contributors

- [x] 2.1 Migrate `AnonymousContributor` — verify test passes
- [x] 2.2 Migrate `Oauth2IntrospectionContributor` — verify test passes
- [x] 2.3 Migrate `AuthCachingContributor` — verify test passes
- [x] 2.4 Migrate `IpCheckOpaContributor` — verify test passes
- [x] 2.5 Migrate `JwtClaimCheckContributor` + `JwtClaimCheckSupport` — verify both tests pass
- [x] 2.6 Migrate `KeycloakRoleCheckContributor` — verify test passes
- [x] 2.7 Migrate `JwtAuthenticationContributor` — verify test passes
- [x] 2.8 Migrate `ApiKeyAuthenticationContributor` — verify test passes
- [x] 2.9 Migrate `AppIdKeyAuthenticationContributor` — verify test passes
- [x] 2.10 Migrate `EmptyAuthenticationContributor` — verify test passes
- [x] 2.11 Remove AuthPolicy builder/merger + 10 contributors from ArchUnit allowlist — verify `ArchitectureTest` passes

## 3. RateLimitPolicy

- [x] 3.1 Rewrite `RateLimitSupport.java` (5 call sites) to build `RateLimitPolicyManifest` records with typed numeric fields — verify `RateLimitSupportTest` passes
- [x] 3.2 Rewrite `RateLimitPolicyGenerator.java` to serialize via `ManifestSerializer` — verify `RateLimitPolicyGeneratorTest` passes
- [x] 3.3 Remove both from ArchUnit allowlist — verify passes

## 4. Simple Kuadrant generators

- [x] 4.1 Rewrite `TlsPolicyGenerator` using `TlsPolicyManifest` record — verify test passes
- [x] 4.2 Rewrite `DnsPolicyGenerator` (2 call sites) using `DnsPolicyManifest` record — verify test passes
- [x] 4.3 Rewrite `ApiProductGenerator` using `ApiProductManifest` record — verify description with embedded quotes round-trips correctly (Jackson handles quoting, `replace("\"", "'")` hack removed) — verify test passes
- [x] 4.4 Rewrite `ApiKeyGenerator` using `ApiKeyManifest` record — verify test passes
- [x] 4.5 Remove all 4 from ArchUnit allowlist — verify passes

## 5. Edge-case / unhappy-path tests

- [x] 5.1 Test `AuthPolicyBuilder.build()` with zero authentication rules and zero authorization rules — verify produces valid manifest with empty sections (not crash or malformed YAML)
- [x] 5.2 Test `AuthPolicyBuilder` with duplicate rule names (e.g. two `addAuthentication("jwt", ...)`) — verify last-write-wins or throws descriptive error (not silent data loss)
- [x] 5.3 Test `AuthPolicyYamlMerger` merging two manifests where one has empty authentication and the other has rules — verify clean merge without null entries
- [x] 5.4 Test `RateLimitSupport` with rate `0`, window `0`, or negative numeric values — verify serializes predictably (Jackson serializes `0`, not omits)
- [x] 5.5 Test `RateLimitPolicyManifest` with multiple limits — verify ordering is deterministic (LinkedHashMap insertion order)
- [x] 5.6 Test `TlsPolicyGenerator` with null/missing `issuerRef` — verify fails fast with clear error message, not a YAML with `issuerRef: null`
- [x] 5.7 Test `ApiProductGenerator` with description containing embedded quotes, newlines, and YAML-special chars (`---`, `:`, `#`) — verify Jackson handles quoting correctly (regression for the `replace("\"", "'")` hack removal)
- [x] 5.8 Test `ApiKeyGenerator` with empty/blank API key value — verify behavior is explicit
- [x] 5.9 Test `JwtClaimCheckSupport` with empty claims list and with claims containing special regex chars — verify structural output is correct
- [x] 5.10 CORS indentation regression test: construct an AuthPolicy with `Access-Control-Allow-Credentials` header in CORS response — assert exact YAML indentation is correct structurally (#53 original report)

## 6. Regression

- [x] 6.1 Run full `ConversionServiceTest` — update `policy.yaml`/`ratelimitpolicy.yaml`/`tlspolicy.yaml`/`dnspolicy.yaml`/`apiproduct.yaml`/`apikey.yaml` assertions — verify CI Linux green
- [x] 6.2 Run `ConversionServiceConcurrencyTest` — verify passes
- [x] 6.3 Verify CORS `Access-Control-Allow-Credentials` indentation bug (#53 original report) structurally cannot recur — the typed record/builder makes the bug class impossible

## Verification

```bash
cd ../migration-toolkit-rhcl/backend && mvn test
cd ../migration-toolkit-rhcl/backend && mvn verify
```

## Verify

- [x] Run `/verify` — record result in `verify-report.md`
