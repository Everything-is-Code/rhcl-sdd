## Why

`GatewayGenerator`, `ConfigMapGenerator`, `SecretGenerator`/`SecretBuilder`, and 5 secret contributors build vanilla Kubernetes resources via `String.formatted()`. Fabric8 ships typed builders for all of them — the lowest-risk tier to migrate first.

## What Changes

- `GatewayGenerator`: build Gateway API `Gateway` via Fabric8 `GatewayBuilder`, serialize through `ManifestSerializer`.
- `ConfigMapGenerator`: build `ConfigMap` via Fabric8 `ConfigMapBuilder`.
- `SecretGenerator` + `SecretBuilder` (contributor accumulator): refactor the shared builder to wrap a Fabric8 `SecretBuilder`, exposing `addStringData(key, value)` for contributors.
- 5 secret contributors (`AnonymousSecretContributor`, `ApiKeySecretContributor`, `AppIdKeySecretContributor`, `DefaultCredentialsSecretContributor`, `TokenIntrospectionSecretContributor`): migrate to call typed builder methods instead of `.formatted()`.
- Each migrated class's tests gain `YamlAssertions.parse(...)` structural checks alongside/replacing `contains()` assertions.
- **Behavior-preserving**: output content unchanged. Field-order diffs (if any) documented in PR.

## Capabilities

### New Capabilities

None — no new user-observable behavior. `skip_specs: true`. All refactored generators and any new internal classes require unit test coverage to maintain Codecov thresholds.

### Modified Capabilities

None — `skip_specs: true`.

## Impact

| Area | Impact |
|------|--------|
| **Backend** | 8 classes rewritten internally; no public method signatures change |
| **Tests** | `GatewayGeneratorTest`, `ConfigMapGeneratorTest`, `SecretGeneratorTest`, `SecretBuilderTest`, 5 secret contributor tests migrated to structural assertions. Codecov must not degrade — new/refactored lines must be covered |
| **Dependencies** | Depends on `typed-yaml-infra` (phase 0). Independent of phases 2/3/4 |
| **GitHub** | Phase 1 of [#262](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/262) |
