## Why

`HttpRouteGenerator`/`HttpRouteBuilder` and 8 contributors build `httproute.yaml` — the output file with the highest contributor fan-out. Fabric8 has a typed Gateway API `HTTPRoute` model. Split from phase 1 due to size and recommended to run last (highest regression risk).

## What Changes

- `HttpRouteGenerator` + `HttpRouteBuilder`: refactor builder to wrap a Fabric8 `HTTPRouteBuilder`, exposing `addRule(HTTPRouteRule)` / `addAnnotation(key, value)` for contributors.
- 8 contributors migrated to build typed Fabric8 objects: `MappingRulesContributor`, `RoutingContributor`, `UpstreamContributor`, `CorsOptionsContributor`, `RetryContributor`, `TimeoutsContributor`, `HeaderModContributor`, `HttpRouteAnnotationsContributor`.
- 3 support classes migrated: `HttpRouteSupport` (URL rewrite filter), `RoutingSupport` (routing body), `UpstreamSupport` (backendRef).
- Contributor `@Priority` ordering preserved exactly so filter/rule order in output is unchanged.
- All serialization through `ManifestSerializer`. Tests migrated to `YamlAssertions`.
- **Behavior-preserving**. `HttpRouteContributorDiscoveryTest` CDI discovery unaffected.
- After this phase: ArchUnit allowlist should be **empty** — closing the #262 epic's structural goal.

## Capabilities

### New Capabilities

None — no new user-observable behavior. `skip_specs: true`. All refactored builders/contributors require unit test coverage to maintain Codecov thresholds.

### Modified Capabilities

None — `skip_specs: true`.

## Impact

| Area | Impact |
|------|--------|
| **Backend** | ~12 classes rewritten internally |
| **Tests** | ~14 test classes; `ConversionServiceTest`'s largest httproute assertion block (~30 of ~100) re-verified. Codecov must not degrade — new/refactored lines must be covered |
| **Dependencies** | Depends on `typed-yaml-infra` (phase 0). Independent of phases 1/2/3 but recommended last |
| **GitHub** | Phase 4 of [#262](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/262). Closing phase — empty ArchUnit allowlist signals epic completion |
