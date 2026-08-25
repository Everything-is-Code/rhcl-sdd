# Tasks — conversion-strategy-registry

GitHub: [#40](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/40)

## Notes for implementers

### ConversionOptions (already exists — do not duplicate)

`ConversionOptions` **already lives** in `dto/ConversionOptions.java` (mutable class with flags: `corsNative`, `retriesSupported`, `includeTlsPolicy`, `ipCheckMode`, etc.). `ConversionService.convert()` already accepts it.

**P1 does not create a new `ConversionOptions`.** Tasks 1.1 audit and extend the existing DTO if any `convert()` overload still bypasses it; keep REST/controller bindings on `dto.ConversionOptions` unless a maintainer explicitly migrates to a record.

### Contributor auto-discovery test (what it means)

Today, adding a policy often requires editing `ConversionService` or a giant `if` chain. After P4, each `HttpRouteContributor` (and Auth/Secret equivalents) is a separate CDI bean. **Auto-discovery** means: register a new contributor class with `@ApplicationScoped` and the parent generator collects it via `Instance<HttpRouteContributor>` — **no edit** to `ConversionService` or `HttpRouteGenerator`.

The **discovery test** proves that: a test-only dummy contributor bean on the classpath is picked up automatically and changes the YAML — without editing the orchestrator or parent generator. That guards the #149 workflow (“new policy = new contributor class only”).

**Test pattern (Quarkus CDI):**

- Use `@QuarkusTest` + `@Inject` on the generator/registry under test (not `new ConversionService()`).
- Place dummy beans in `src/test/java/.../discovery/` as `@ApplicationScoped` implementations with a unique marker string (e.g. annotation `x-discovery-marker: rhcl-test`).
- Use `@Priority` on the dummy bean if ordering matters; production contributors keep default priority.
- Each discovery test class lives beside its target: `.../generator/ResourceGeneratorRegistryDiscoveryTest.java`, `.../contributor/HttpRouteContributorDiscoveryTest.java`, etc.
- **Negative case:** assert the marker is absent when the dummy bean is not registered (plain unit test or `@QuarkusTest` without the test profile).

---

## Prerequisites

- [x] 0.1 Review `proposal.md`, `design.md`, and `specs/conversion-pipeline/spec.md` in this change
- [x] 0.2 Create feature branch in product repo: `feature/conversion-strategy-registry` from current `main`
- [x] 0.3 Confirm CI green on `main` before starting (`cd backend && mvn test`) — local: 2 CORS indent tests fail on Windows CRLF (`\n` vs `\r\n`); trust CI Linux

## 1. P1 — Spine (shared utilities)

- [x] 1.1 Audit `dto/ConversionOptions.java`: list every `convert()` overload and confirm all paths pass options through; add any missing fields still carried only as positional args — verify `mvn -q -DskipTests compile` and existing `ConversionServiceTest` option-related tests pass
- [x] 1.2 Add `PolicyFinder` with `findEnabled(ApiService, String policyName)` in `service/PolicyFinder.java` — verify `PolicyFinderTest` for present, absent, and disabled policy chain entries
- [x] 1.3 Extract `toKebabCase` to `util/StringUtils.java` and update `ConversionService` to use it — verify `StringUtilsTest` parameterized cases pass
- [x] 1.4 Replace all 12 `find*Policy` private methods in `ConversionService` with `PolicyFinder` calls — verify `mvn test -Dtest=ConversionServiceTest` passes on CI Linux

## 2. P2 — Orchestration helpers (extract before registry loop)

- [x] 2.1 Move `ResolvedBackend` and `resolveBackends` / `resolveOne` to `service/conversion/ResolvedBackend.java` and `service/conversion/BackendResolver.java` (or equivalent); inject resolver into `ConversionService` — verify backend-resolution tests in `ConversionServiceTest` (multi-backend, external URL override ignored) pass
- [x] 2.2 Move `emitDnsPolicy(ConversionOptions)` and primary-backend helpers (`primaryType`, `primaryExternalHost`, `overrideIgnored` warning logic) into `service/conversion/ConversionContext.java` built once per `convert()` call — verify DNS/TLS option tests pass
- [x] 2.3 Move URL helpers `extractHostname`, `extractInternalService`, `extractPort` to `util/UrlUtils.java` (or keep in resolver package) — verify unit tests for hostname/port extraction pass
- [x] 2.4 Move `parseJsonObjectConfig` and `stripMigratedFromLabel` to `service/conversion/ConversionYamlSupport.java` (or `util`) — verify logging/url-rewriting EnvoyFilter tests pass
- [x] 2.5 Move `resolveRetryAttempts`, `resolveContentLimitBytes` to shared policy config helpers used by generators — verify retry and content-limits tests pass

## 3. P2 — Registry spine (passthrough generators)

- [x] 3.1 Add `ResourceGenerator` interface (`outputKey()`, `applies(ConversionContext)`, `generate(...)`) and `ResourceGeneratorRegistry` (CDI `Instance<ResourceGenerator>`) under `service/generator/` — verify registry unit test resolves by output key
- [x] 3.1b Add `ResourceGeneratorRegistryDiscoveryTest` (`@QuarkusTest`): dummy `@ApplicationScoped` `ResourceGenerator` in `src/test/java/.../discovery/TestMarkerGatewayGenerator.java` with `outputKey()` `gateway.yaml` appends marker; inject registry, call `generateAll()`, assert marker in `gateway.yaml` — verify test passes without editing `ResourceGeneratorRegistry` source
- [x] 3.2 Passthrough `GatewayGenerator` → `gateway.yaml` — verify generator test matches current `gateway.yaml` assertions in `ConversionServiceTest`
- [x] 3.3 Passthrough `HttpRouteGenerator` → `httproute.yaml` (delegates to existing `generateHttpRoute` until P4) — verify httproute tests pass
- [x] 3.4 Passthrough `AuthPolicyGenerator` → `policy.yaml` — verify policy.yaml tests pass
- [x] 3.5 Passthrough `SecretGenerator` → `secret.yaml` — verify secret.yaml tests pass
- [x] 3.6 Passthrough `ConfigMapGenerator` → `configmap.yaml` — verify configmap tests pass
- [x] 3.7 Passthrough `ApiProductGenerator` → `apiproduct.yaml` and `ApiKeyGenerator` → `apikey.yaml` (when `authentication.type == apiKey`) — verify apiproduct/apikey tests pass
- [x] 3.8 Passthrough `ServiceEntryGenerator` → `serviceentry.yaml` and `DestinationRuleGenerator` → `destinationrule.yaml` (external backends only) — verify serviceentry/destinationrule tests pass
- [x] 3.9 Passthrough `TelemetryGenerator` → `telemetry.yaml` (when logging policy present) — verify telemetry tests pass
- [x] 3.10 Passthrough `LoggingEnvoyFilterGenerator` → `envoyfilter-logging.yaml` (when json_object_config non-empty) — verify logging EnvoyFilter tests pass
- [x] 3.11 Passthrough `UrlRewritingEnvoyFilterGenerator` → `envoyfilter-url-rewriting.yaml` (when url_rewriting commands present) — verify url-rewriting tests pass
- [x] 3.12 Passthrough `ContentLimitsEnvoyFilterGenerator` → `envoyfilter-content-limits.yaml` (when request bytes > 0) — verify content-limits tests pass
- [x] 3.13 Passthrough `RetryEnvoyFilterGenerator` → `envoyfilter-retry.yaml` (when `!retriesSupported` and retry policy present) — verify retry EnvoyFilter tests pass
- [x] 3.14 Passthrough `AuthorizationPolicyGenerator` → `authorizationpolicy.yaml` (when ip_check + `ipCheckMode == authorizationPolicy`) — verify authorizationpolicy tests pass
- [x] 3.15 Passthrough `RateLimitPolicyGenerator` → `ratelimitpolicy.yaml` (when edge limiting yields YAML) — verify ratelimitpolicy tests pass
- [x] 3.16 Passthrough `TlsPolicyGenerator` → `tlspolicy.yaml` and `DnsPolicyGenerator` → `dnspolicy.yaml` (when `includeTlsPolicy` / `emitDnsPolicy`) — verify TLS/DNS option tests pass
- [x] 3.17 Passthrough `ReadmeGenerator` → `README.md` (delegates to existing `generateReadme` — no new positional args, #170) — verify README tests pass
- [x] 3.18 Refactor `ConversionService.convert()` to build `ConversionContext` once and invoke `ResourceGeneratorRegistry` for applicable generators (conditional `applies()` mirrors current `if` branches) — verify full `ConversionServiceTest` suite passes on CI Linux
- [x] 3.19 Extend `ArchitectureTest` for `service.generator` and `service.generator.contributor` layering — verify `mvn test -Dtest=ArchitectureTest` passes
- [x] 3.20 **Merge P2 to `main` early** — deferred; maintainer will merge single end-to-end PR for #40 when verify + review complete

## 4. P3 — Move logic into simple generators (no contributors)

- [x] 4.1 Move implementation into `GatewayGenerator` (incl. listener hostnames from DNS opts) — verify gateway tests; delete passthrough delegation
- [x] 4.2 Move implementation into `DestinationRuleGenerator`, `ServiceEntryGenerator` — verify tests; delete private `generateDestinationRule` / `generateServiceEntry`
- [x] 4.3 Move implementation into `TelemetryGenerator`, `LoggingEnvoyFilterGenerator` — verify telemetry/logging tests
- [x] 4.4 Move implementation into `ConfigMapGenerator`, `RateLimitPolicyGenerator` — verify configmap/ratelimit tests
- [x] 4.5 Move implementation into `ApiProductGenerator`, `ApiKeyGenerator` — verify apiproduct/apikey tests
- [x] 4.6 Move implementation into all four EnvoyFilter generators (`UrlRewriting`, `ContentLimits`, `Retry`) — verify respective EnvoyFilter tests
- [x] 4.7 Move implementation into `TlsPolicyGenerator`, `DnsPolicyGenerator`, `AuthorizationPolicyGenerator` — verify TLS/DNS/authorizationpolicy tests
- [x] 4.8 Confirm `ConversionService.java` line count dropped vs pre-P3 — verify `mvn test -Dtest=ConversionServiceTest`

## 5. P4 — HTTPRoute contributor infrastructure

- [x] 5.1 Add `HttpRouteBuilder` and `HttpRouteContributor` interface under `service/generator/contributor/` — verify builder unit test assembles route fragments
- [x] 5.2 Refactor `HttpRouteGenerator` to apply all CDI `Instance<HttpRouteContributor>` beans in stable order — verify full httproute regression tests pass
- [x] 5.3a Add test dummy `TestMarkerHttpRouteContributor` in `src/test/java/.../discovery/` (`@ApplicationScoped`, `@Priority(999)`) that appends `x-discovery-marker: rhcl-httproute-test` to `HttpRouteBuilder` — verify compiles
- [x] 5.3b Add `HttpRouteContributorDiscoveryTest` (`@QuarkusTest`, inject `HttpRouteGenerator`): generate with minimal `ApiService` fixture; assert output contains `rhcl-httproute-test` — verify passes without editing `HttpRouteGenerator` or `ConversionService`
- [x] 5.3c Add negative test: same generator setup with dummy bean excluded (or `@QuarkusTestProfile` without test beans); assert marker absent — verify `HttpRouteContributorDiscoveryTest` negative case passes
- [x] 5.4 Extract `MappingRulesContributor` (path rules, multi-backend `buildRuleFiltersBlock`) — verify mapping-rules and multi-backend httproute tests pass
- [x] 5.5 Extract `MultiBackendContributor` (per-backend rules, shared filters) — verify multi-backend routing tests pass (via `MappingRulesContributor` + `HttpRouteSupport.selectBackendsForPath`)
- [x] 5.6 Extract `HeaderModContributor` (`buildHeaderModificationFilters`) — verify header_modification tests pass
- [x] 5.7 Extract `CorsContributor` (`buildCorsFilters`, `corsNative` flag) — verify cors native vs ResponseHeaderModifier tests pass (`CorsFiltersContributor` + `CorsOptionsContributor`)
- [x] 5.8 Extract `UrlRewriteContributor` for HTTPRoute-native rewrite fragments (if any remain in httproute; Envoy path stays in UrlRewritingEnvoyFilterGenerator) — verify url_rewriting httproute vs envoyfilter split tests pass (`HttpRouteSupport.buildRuleFiltersBlock`)
- [x] 5.9 Extract `RetryContributor` (`buildRetryBlock` when `retriesSupported`) — verify HTTPRoute retry.attempts tests pass
- [x] 5.10 Extract `TimeoutsContributor` (`buildTimeoutsBlock`) — verify timeout annotation/filter tests pass
- [x] 5.11 Extract `HttpRouteAnnotationsContributor` (`buildHttpRouteAnnotations`, `buildUpstreamAnnotations`) — verify annotation-related tests pass
- [x] 5.12 Define contributor order (`@Priority` or explicit sort) matching pre-refactor YAML fragment order — verify no httproute assertion drift on CI

## 6. P5 — AuthPolicy contributors (policy.yaml)

- [x] 6.1 Add `AuthPolicyBuilder`, `AuthPolicyContributor`, refactor `AuthPolicyGenerator` to use `Instance<AuthPolicyContributor>` — verify builder tests pass
- [x] 6.2a Add test dummy `TestMarkerAuthPolicyContributor` in `src/test/java/.../discovery/` appending `x-discovery-marker: rhcl-authpolicy-test` to `AuthPolicyBuilder` — verify compiles
- [x] 6.2b Add `AuthPolicyContributorDiscoveryTest` (`@QuarkusTest`, inject `AuthPolicyGenerator`): assert marker in `policy.yaml` output — verify without editing `AuthPolicyGenerator` or `ConversionService`
- [x] 6.2c Add negative case (marker absent without dummy bean) — verify `AuthPolicyContributorDiscoveryTest` negative case passes
- [x] 6.3 Extract `AnonymousContributor` (`generateAnonymousAuthPolicy`, `anonymousTarget` option) — verify anonymous access tests pass
- [x] 6.4 Extract `Oauth2IntrospectionContributor` (`generateOauth2IntrospectionAuthPolicy`) — verify token_introspection policy tests pass
- [x] 6.5 Extract `AuthCachingContributor` (`buildAuthCacheBlock` composed with introspection) — verify auth_caching tests pass
- [x] 6.6 Extract `IpCheckOpaContributor` (`appendIpCheckOpaIfNeeded`, `buildIpCheckOpaAuthorization`, `ipCheckMode == authPolicyOpa`) — verify ip_check OPA tests pass
- [x] 6.7 Extract `JwtClaimCheckContributor` (`appendJwtClaimCheckAuthorization`, `buildJwtClaimCheckNamedRule`) — verify jwt_claim_check tests pass
- [x] 6.8 Extract `KeycloakRoleCheckContributor` (`appendKeycloakRoleCheckAuthorization`) — verify keycloak_role_check tests pass
- [x] 6.9 Move `finalizeAuthPolicyAuthorization` / `mergeAuthorizationNamedRules` into builder or a shared `AuthPolicyYamlMerger` helper — verify composite auth policy tests pass
- [x] 6.10 Delete private `generateAuthPolicy` body from `ConversionService` — verify all policy.yaml tests pass

## 7. P5 — Secret contributors (secret.yaml)

- [x] 7.1 Add `SecretBuilder`, `SecretContributor`, refactor `SecretGenerator` to use `Instance<SecretContributor>` — verify builder tests pass
- [x] 7.2a Add test dummy `TestMarkerSecretContributor` in `src/test/java/.../discovery/` appending `x-discovery-marker: rhcl-secret-test` to `SecretBuilder` — verify compiles
- [x] 7.2b Add `SecretContributorDiscoveryTest` (`@QuarkusTest`, inject `SecretGenerator`): assert marker in `secret.yaml` output — verify without editing `SecretGenerator` or `ConversionService`
- [x] 7.2c Add negative case (marker absent without dummy bean) — verify `SecretContributorDiscoveryTest` negative case passes
- [x] 7.3 Extract `AnonymousSecretContributor` (anonymous policy credentials) — verify anonymous secret tests pass
- [x] 7.4 Extract `TokenIntrospectionSecretContributor` (`generateTokenIntrospectionSecret`) — verify introspection secret tests pass
- [x] 7.5 Extract `AppIdKeySecretContributor` (`generateAppIdKeySecret`, api_key auth) — verify app_id/app_key secret tests pass
- [x] 7.6 Delete private `generateSecret` body from `ConversionService` — verify all secret.yaml tests pass

## 8. P6 — Cleanup and hardening

- [x] 8.1 Delete orphan private `generate*` and helper methods from `ConversionService`; target ~100–150 lines orchestrator — verify line count and full test suite pass
- [x] 8.2 Add concurrent conversion test: two threads, different `ApiService` inputs, no shared mutable state — verify `ConversionServiceConcurrencyTest` passes
- [x] 8.3 Remove unnecessary `@SuppressWarnings` where typed models suffice — verify `mvn verify` passes (Checkstyle, PMD, ArchUnit, JaCoCo)
- [x] 8.4 Confirm no private `find*Policy` remains outside `PolicyFinder` — verify grep/architecture check in PR description
- [x] 8.5 Add ArchUnit rule: classes in `..contributor..` MUST NOT depend on `ConversionService` (contributors register via CDI only) — verify `ArchitectureTest` passes

## 9. Discovery tests — summary checklist

All discovery tests use `@QuarkusTest` + test-only beans in `src/test/java/com/redhat/migrationtoolkit/rhcl/service/generator/discovery/`.

| Test class | Injects | Dummy bean | Marker | Phase |
|------------|---------|------------|--------|-------|
| `ResourceGeneratorRegistryDiscoveryTest` | `ResourceGeneratorRegistry` | `TestMarkerGatewayGenerator` | in `gateway.yaml` | P2 (3.1b) |
| `HttpRouteContributorDiscoveryTest` | `HttpRouteGenerator` | `TestMarkerHttpRouteContributor` | `rhcl-httproute-test` | P4 (5.3b–c) |
| `AuthPolicyContributorDiscoveryTest` | `AuthPolicyGenerator` | `TestMarkerAuthPolicyContributor` | `rhcl-authpolicy-test` | P5 (6.2b–c) |
| `SecretContributorDiscoveryTest` | `SecretGenerator` | `TestMarkerSecretContributor` | `rhcl-secret-test` | P5 (7.2b–c) |

- [x] 9.3 Run full discovery suite: `mvn test -Dtest=*DiscoveryTest` — verify 4 test classes, each with positive + negative case, all green on CI Linux

## Verification

```bash
cd ../migration-toolkit-rhcl/backend && mvn test
cd ../migration-toolkit-rhcl/backend && mvn verify
cd ../migration-toolkit-rhcl/backend && mvn test -Dtest=ConversionServiceTest
cd ../migration-toolkit-rhcl/backend && mvn test -Dtest=ArchitectureTest
cd ../migration-toolkit-rhcl/backend && mvn test -Dtest=*DiscoveryTest
```

- Windows CRLF: if local `ConversionServiceTest` fails on YAML whitespace only, confirm GitHub Actions Linux CI before treating as regression.
- REST smoke: convert/preview endpoints return same status and body shape as pre-refactor.

## Docs

- [x] 10.1 Update `docs/conversion-architecture.md` in SDD store to list all output files (incl. EnvoyFilters, apiproduct, apikey, optional TLS/DNS) — verify doc matches implemented layout
- [x] 10.2 Update `docs/sdd-backlog.md` row for `conversion-strategy-registry` — status reflects implementation pending merge; full closure on `/opsx-archive` after PR merge

## Verify

- [x] Run `/verify` — see `verify-report.md` in this change folder (PASS 2026-08-25, re-run)
