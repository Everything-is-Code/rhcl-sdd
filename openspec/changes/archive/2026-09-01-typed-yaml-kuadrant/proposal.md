## Why

`AuthPolicyGenerator` (+ 10 contributors), `RateLimitPolicyGenerator`/`RateLimitSupport`, `TlsPolicyGenerator`, `DnsPolicyGenerator`, `ApiProductGenerator`, and `ApiKeyGenerator` build Kuadrant CRD YAML via `String.formatted()`. No Fabric8 typed model exists for these CRDs. This phase uses the hand-written Java records from phase 0's `model/kuadrant/` package to replace string templates with typed constructors — giving compile-time safety without codegen or upstream coupling.

This is the largest phase (~18 classes, ~24 call sites) and the highest-value target: `AuthPolicy` generation is where the original CORS indentation bug occurred.

## What Changes

- `AuthPolicyGenerator` + `AuthPolicyBuilder` + `AuthPolicyYamlMerger`: refactor builder to accumulate typed section records (`AuthenticationRule`, `AuthorizationRule`, etc.) instead of string fragments; produce an `AuthPolicyManifest` record via `builder.build()`; serialize via `ManifestSerializer`.
- 10 AuthPolicy contributors: call typed builder methods (`builder.addAuthentication("jwt", new JwtAuth(...))`) instead of `.formatted()`.
- `RateLimitPolicyGenerator` + `RateLimitSupport` (5 call sites): construct `RateLimitPolicyManifest` records.
- `TlsPolicyGenerator`, `DnsPolicyGenerator`, `ApiProductGenerator`, `ApiKeyGenerator`: construct their respective manifest records directly.
- `JwtClaimCheckSupport`: migrate fragment-building from `.formatted()` to typed record construction.
- **Behavior-preserving**: semantic output unchanged for structural fields. RateLimitPolicy inline `# WARNING:` YAML comments are intentionally dropped per `design.md` decision #5 (warnings remain in README via `ReadmeSupport`). `AuthPolicyContributorDiscoveryTest` CDI discovery unaffected.

## Capabilities

### New Capabilities

None — no new user-observable behavior. `skip_specs: true`. All new record types and refactored generators/builders require unit test coverage to maintain Codecov thresholds.

### Modified Capabilities

None — `skip_specs: true`.

## Impact

| Area | Impact |
|------|--------|
| **Backend** | ~18 classes rewritten internally. AuthPolicy builder changes from string-merge to typed-record accumulation — the most significant internal refactor in the epic |
| **Tests** | ~15 test classes migrated to structural assertions; `ConversionServiceTest`'s largest assertion block (~40 of ~100 for `policy.yaml`) re-verified. New record types need their own unit tests (`*ManifestTest`). Codecov must not degrade |
| **Dependencies** | Depends on `typed-yaml-infra` (phase 0, `model/kuadrant/` records). Independent of phases 1/2/4 |
| **GitHub** | Phase 3 of [#262](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/262) |
