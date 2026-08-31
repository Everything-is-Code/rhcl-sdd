# Tasks — typed-yaml-httproute

GitHub: [#262](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/262) — Phase 4 (recommended last)

## Prerequisites

- [ ] 0.1 Confirm `typed-yaml-infra` (phase 0) is merged
- [ ] 0.2 Rebase/create feature branch: `feature/262-typed-yaml-httproute` from latest `main` (ideally after phases 1-3 merged)
- [ ] 0.3 Confirm CI green on `main`

## 1. Shared builder

- [ ] 1.1 Rewrite `HttpRouteBuilder.java` to wrap a Fabric8 `HTTPRouteBuilder`, exposing `addRule(HTTPRouteRule)` / `addAnnotation(key, value)` — verify `HttpRouteBuilderTest` passes
- [ ] 1.2 Rewrite `HttpRouteGenerator.java` to serialize via `ManifestSerializer.toYaml(...)` — verify `HttpRouteGeneratorTest` passes
- [ ] 1.3 Confirm `HttpRouteContributorDiscoveryTest` still passes unmodified

## 2. Support classes

- [ ] 2.1 Migrate `HttpRouteSupport.java` — return typed `HTTPRouteFilter`/`URLRewrite` objects instead of YAML strings — verify `HttpRouteSupportTest` passes
- [ ] 2.2 Migrate `RoutingSupport.java` — return typed rule/match objects — verify `RoutingSupportTest` passes
- [ ] 2.3 Migrate `UpstreamSupport.java` — return typed `HTTPBackendRef` objects — verify `UpstreamSupportTest` passes

## 3. Contributors — matching and routing

- [ ] 3.1 Migrate `MappingRulesContributor` (2 call sites) — verify test passes
- [ ] 3.2 Migrate `RoutingContributor` — verify test passes
- [ ] 3.3 Migrate `UpstreamContributor` — verify test passes

## 4. Contributors — filters

- [ ] 4.1 Migrate `CorsOptionsContributor` — verify test passes
- [ ] 4.2 Migrate `RetryContributor` — verify test passes
- [ ] 4.3 Migrate `TimeoutsContributor` — verify test passes (note: `CorsFiltersContributor` test also covers CORS filter path)
- [ ] 4.4 Migrate `HeaderModContributor` — verify test passes
- [ ] 4.5 Migrate `HttpRouteAnnotationsContributor` (annotations map, mechanical) — verify test passes
- [ ] 4.6 Verify `@Priority` ordering unchanged: compare contributor `@Priority` values against #40 baseline — add/confirm test asserting filter/rule array order matches expected sequence
- [ ] 4.7 Remove all 12 classes (generator, builder, 8 contributors, 3 support) from ArchUnit allowlist — verify `ArchitectureTest` passes
- [ ] 4.8 Confirm ArchUnit allowlist is now **empty** — note in PR description as epic-closing signal

## 5. Edge-case / unhappy-path tests

- [ ] 5.1 Test `HttpRouteBuilder` with zero rules added (no contributors contribute) — verify produces valid HTTPRoute with empty `rules: []` (not crash)
- [ ] 5.2 Test `HttpRouteBuilder` with duplicate `addAnnotation(key, ...)` calls for the same key — verify last-write-wins behavior
- [ ] 5.3 Test `MappingRulesContributor` with service that has no mapping rules — verify contributor is a no-op (does not add empty rules)
- [ ] 5.4 Test `UpstreamContributor` with null/empty upstream URL — verify throws descriptive error or skips cleanly
- [ ] 5.5 Test `CorsOptionsContributor` with empty allowed-origins list — verify produces valid CORS filter (not a filter with `null` values)
- [ ] 5.6 Test `HeaderModContributor` with header name containing invalid HTTP header chars — verify behavior (reject or sanitize)
- [ ] 5.7 Test `TimeoutsContributor` with zero timeout value — verify serializes as `0s` or equivalent (not omitted)
- [ ] 5.8 Test `HttpRouteSupport` URL rewrite with empty/null rewrite path — verify predictable behavior
- [ ] 5.9 Test `@Priority` ordering: add two contributors with same `@Priority` value — verify stable output order (no flaky test from non-deterministic ordering)
- [ ] 5.10 Test `RoutingSupport` with path match containing regex-special chars — verify not interpreted as regex by the builder

## 6. Regression

- [ ] 6.1 Run full `ConversionServiceTest` — update httproute assertions (~30 of ~100) — verify CI Linux green
- [ ] 6.2 Run `ConversionServiceConcurrencyTest` — verify passes

## Verification

```bash
cd ../migration-toolkit-rhcl/backend && mvn test
cd ../migration-toolkit-rhcl/backend && mvn verify
```

## Docs

- [ ] 7.1 Once merged, comment on #262 that all 5 phases are complete — maintainer decides whether to close #262

## Verify

- [ ] Run `/verify` — record result in `verify-report.md`
