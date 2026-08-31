## Why

9 generators produce Istio resources via `String.formatted()` even though Fabric8 ships typed Istio models. Independent of core K8s (phase 1) and Kuadrant (phase 3) — grouping them lets the Istio-model work land as one cohesive PR.

## What Changes

- `ServiceEntryGenerator`, `DestinationRuleGenerator`: migrate to Fabric8 Istio `ServiceEntryBuilder`/`DestinationRuleBuilder`.
- 5 EnvoyFilter generators (`Retry`, `Logging`, `ContentLimits`, `UrlRewriting`, `MaintenanceMode`): migrate to Fabric8 `EnvoyFilterBuilder`. Embedded Lua scripts remain string values — only the outer YAML envelope changes.
- `AuthorizationPolicyGenerator` (Istio, distinct from Kuadrant `AuthPolicy`): migrate to Fabric8 `AuthorizationPolicyBuilder`.
- `TelemetryGenerator`: migrate to Fabric8 `TelemetryBuilder`.
- All serialization through `ManifestSerializer`. Tests migrated to `YamlAssertions`.
- **Behavior-preserving**: no output content change.

## Capabilities

### New Capabilities

None — no new user-observable behavior. `skip_specs: true`. All refactored generators require unit test coverage to maintain Codecov thresholds.

### Modified Capabilities

None — `skip_specs: true`.

## Impact

| Area | Impact |
|------|--------|
| **Backend** | 9 generator classes rewritten internally |
| **Tests** | 9 generator test classes migrated to structural assertions. Codecov must not degrade — new/refactored lines must be covered |
| **Dependencies** | Depends on `typed-yaml-infra` (phase 0). Independent of phases 1/3/4 |
| **GitHub** | Phase 2 of [#262](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/262) |
