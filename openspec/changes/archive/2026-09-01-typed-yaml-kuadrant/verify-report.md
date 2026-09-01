# Verify report — typed-yaml-kuadrant

**Change:** `typed-yaml-kuadrant`  
**Issue:** [#262](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/262) (phase 3)  
**Branch:** `feature/262-typed-yaml-kuadrant` (stacked on `feature/262-typed-yaml-istio`)  
**Verified:** 2026-09-01

## Scope

| Flag | Value |
|------|-------|
| `BACKEND_TOUCHED` | yes |
| `FRONTEND_TOUCHED` | no |

## OpenSpec

| Check | Result |
|-------|--------|
| `tasks.md` all `[x]` | **PASS** (all sections including Verify) |

## Tests

| Command | Result |
|---------|--------|
| `cd backend && mvn test` | **PASS** — exit code 0, all tests green |
| `/verify-fe` | not required (no frontend changes in this phase) |

## Review findings addressed

| Finding | Resolution |
|---------|-----------|
| Major: `proposal.md` "behavior-preserving" vs RateLimit comment removal | `proposal.md` updated — explicitly states comments are intentionally dropped per design #5 |
| Major: nested AuthPolicy rule bodies untyped `Map<String,Object>` | Accepted as follow-up (future sub-phase); core typed migration delivered |
| Moderate: `AuthPolicyYamlMerger` production-dead code | Accepted as follow-up; not deleted to preserve test coverage of typed merger semantics |
| Moderate: task 5.6 TlsPolicy test name contradicts intent | Test renamed to `tlsPolicyGenerator_blankIssuerKindAndName_appliesDefaults` — validates default behavior |
| Moderate: task 5.7 multiline/`---` description not tested | `KuadrantGeneratorEdgeCaseTest.apiProductGenerator_multilineDescriptionAndYamlSeparator_jacksonHandles` added |
| Moderate: task 5.8 ApiKey blank value not covered | `KuadrantGeneratorEdgeCaseTest.apiKeyGenerator_blankSystemName_usesKebabFallback` added |
| Moderate: task 5.9 JwtClaimCheck regex chars incomplete | `JwtClaimCheckSupportTest.buildNamedRule_regexSpecialChars_roundTripsInYaml` added with `.*(?=read)` pattern |
| Moderate: RateLimit equivalence narrow | Leaky-bucket and plan-ceiling paths added to `KuadrantManifestEquivalenceTest` |
| Moderate: no `/verify` artifact | This report created; tasks.md Verify section checked |
| Minor: `includeMigratedFromLabel` inconsistency | `KuadrantManifestSupport.baseLabels(name, flag)` introduced; generators migrated to use it |
| Minor: duplicate rule name overwrite silent | `AuthPolicyBuilder` emits `LOG.debugf(...)` on overwrite |
| Minor: `RateLimitSupport.generateRateLimitPolicy` uses `new ManifestSerializer()` | Accepted as minor; CDI path uses injected serializer |
| Nit: `build_emptyAuthentication_serializesAsEmptyMap` loose assertion | `AuthPolicyBuilderTest` now asserts exact `authentication: {}` |
| Nit: `KuadrantManifestSupportTest` missing | `KuadrantManifestSupportTest` added covering `baseLabels`, `meta`, `resolveSerializer` |
| Additional: negative RateLimit test (task 5.4) | `RateLimitSupportEdgeCaseTest.buildManifest_negativeRate_skipsLimiter` added |

## Spec compliance

| Design goal | Evidence |
|-------------|----------|
| AuthPolicy builder typed accumulator | `AuthPolicyBuilder` + `AuthPolicyBuilderTest` + edge cases |
| 10 contributors migrated from `String.formatted()` | All contributors use `addAuthentication`/`addAuthorization` typed methods |
| RateLimitPolicy typed records | `RateLimitSupport.buildManifest` + `RateLimitSupportEdgeCaseTest` |
| Simple Kuadrant generators typed | `TlsPolicy`, `DnsPolicy`, `ApiProduct`, `ApiKey` generators + tests |
| ArchUnit allowlist shrank by 14+ classes | `ArchitectureTest` (AuthPolicy + RateLimit + simple generators removed) |
| CORS `Access-Control-Allow-Credentials` structurally correct | Typed `HeaderEntry`/`ResponseConfig` + `AuthPolicyBuilderEdgeCaseTest` 5.10 |
| RateLimit inline `# WARNING:` comments intentionally removed | Design #5; README warnings preserved via `ReadmeSupport`; `proposal.md` updated |

## Outcome

**PASS** — ready for commit → stacked PR → `/opsx-archive`.
