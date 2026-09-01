# Tasks — typed-yaml-istio

GitHub: [#262](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/262) — Phase 2

## Prerequisites

- [x] 0.1 Confirm `typed-yaml-infra` (phase 0) is merged — Fabric8 Istio model dep available *(stacked on `feature/262-typed-yaml-k8s-gateway` / PRs #263+#264; infra present on branch)*
- [x] 0.2 Create feature branch: `feature/262-typed-yaml-istio` from current `main`
- [x] 0.3 Confirm CI green on `main` *(deferred to stacked PR CI; local `mvn test` + `mvn verify` green)*

## 1. ServiceEntry / DestinationRule

- [x] 1.1 Rewrite `ServiceEntryGenerator` via `ServiceEntryBuilder` — verify `ServiceEntryGeneratorTest` passes
- [x] 1.2 Rewrite `DestinationRuleGenerator` via `DestinationRuleBuilder` (2 call sites) — verify `DestinationRuleGeneratorTest` passes
- [x] 1.3 Migrate both test classes to structural assertions — verify passes
- [x] 1.4 Remove both from ArchUnit allowlist — verify `ArchitectureTest` passes

## 2. EnvoyFilters

- [x] 2.1 Rewrite `RetryEnvoyFilterGenerator` via `EnvoyFilterBuilder` — verify test passes
- [x] 2.2 Rewrite `LoggingEnvoyFilterGenerator` (incl. `jsonFormat` config patch) — verify test passes
- [x] 2.3 Rewrite `ContentLimitsEnvoyFilterGenerator` — verify test passes
- [x] 2.4 Rewrite `UrlRewritingEnvoyFilterGenerator` (2 call sites) — verify test passes
- [x] 2.5 Rewrite `MaintenanceModeEnvoyFilterGenerator` (2 call sites; Lua script stays string, envelope moves to builder) — verify test passes
- [x] 2.6 Migrate all 5 test classes to structural assertions (keep string checks for Lua content) — verify pass
- [x] 2.7 Remove all 5 from ArchUnit allowlist — verify `ArchitectureTest` passes

## 3. AuthorizationPolicy / Telemetry

- [x] 3.1 Rewrite `AuthorizationPolicyGenerator` via Fabric8 Istio `AuthorizationPolicyBuilder` — verify test passes
- [x] 3.2 Rewrite `TelemetryGenerator` via `TelemetryBuilder` — verify test passes
- [x] 3.3 Migrate both tests to structural assertions — verify passes
- [x] 3.4 Remove both from ArchUnit allowlist — verify `ArchitectureTest` passes

## 4. Edge-case / unhappy-path tests

- [x] 4.1 Test `ServiceEntryGenerator` with a service that has no backends/upstream hosts — verify produces valid ServiceEntry or throws descriptive error
- [x] 4.2 Test `DestinationRuleGenerator` with null/empty TLS mode — verify defaults are applied cleanly (not `null` in YAML)
- [x] 4.3 Test `MaintenanceModeEnvoyFilterGenerator` with Lua script containing YAML-special chars (`---`, `:`, `#`) — verify Lua content is preserved as-is inside a YAML block scalar (not escaped or corrupted)
- [x] 4.4 Test `LoggingEnvoyFilterGenerator` with empty `jsonFormat` config — verify produces valid EnvoyFilter with sensible default or clear error
- [x] 4.5 Test `ContentLimitsEnvoyFilterGenerator` with zero/negative limits — verify behavior (reject or serialize as `0`)
- [x] 4.6 Test `UrlRewritingEnvoyFilterGenerator` with empty rewrite path — verify behavior (reject or produce empty rewrite, not crash)
- [x] 4.7 Test EnvoyFilter builders produce valid YAML even when optional `match` criteria are absent — verify no null-field noise in output

## 5. Regression

- [x] 5.1 Run full `ConversionServiceTest` — update Istio-related string assertions that differ in field order — verify CI Linux green
- [x] 5.2 Run `ConversionServiceConcurrencyTest` — verify passes

## Verification

```bash
cd ../migration-toolkit-rhcl/backend && mvn test
cd ../migration-toolkit-rhcl/backend && mvn verify
```

## Verify

- [x] Run `/verify` — record result in `verify-report.md`
