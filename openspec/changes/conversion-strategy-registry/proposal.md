## Why

`ConversionService.java` has grown to **3223 lines** with 13 `generate*()` methods, 12 `find*Policy()` helpers, and hardcoded orchestration in `convert()`. Every new policy from epic #149 increases merge conflicts and blocks parallel work (#169). The agreed Strategy + Registry + Contributor architecture (issue #40) must land before more policy conversions accumulate in the god class.

## What Changes

- Introduce **Level 1 — ResourceGenerator + Registry**: one generator per output file (`gateway.yaml`, `httproute.yaml`, `policy.yaml`, `secret.yaml`, etc.), looked up via CDI registry instead of inline `convert()` branches.
- Introduce **Level 2 — Contributor pattern** for multi-policy resources: HTTPRoute, AuthPolicy, and Secret aggregate fragments from independent 3scale policies via shared builders.
- Extract shared utilities: `ConversionOptions` record, `PolicyFinder.findEnabled()`, `StringUtils.toKebabCase()`.
- Reduce `ConversionService` to a thin orchestrator (~100–150 lines) that delegates to the registry.
- **Behavior-preserving**: no change to generated YAML output, REST API contracts, or frontend behavior.
- Phased delivery (P1–P6) so #149 policy PRs can target the new structure incrementally.

## Capabilities

### New Capabilities

- `conversion-pipeline`: Structural and behavioral contracts for the 3scale → Kuadrant/Gateway API YAML conversion subsystem — output preservation, concurrency safety, modular extensibility for policy contributions, and unchanged REST conversion API.

### Modified Capabilities

<!-- No existing openspec/specs yet — first baseline capability for conversion -->

## Impact

| Area | Impact |
|------|--------|
| **Backend** | `ConversionService.java` decomposed into `service/generator/` and `service/generator/contributor/` packages; new `PolicyFinder`, `ConversionOptions`, `StringUtils`; `ArchitectureTest` rules updated |
| **Tests** | `ConversionServiceTest` remains regression oracle; new unit tests per generator/contributor; ArchUnit layer rules |
| **API** | No endpoint or DTO changes — `ConversionController` unchanged |
| **Frontend** | None |
| **Dependencies** | Unblocks #169 (stateless generators enable parallel bulk convert); coordinates with #149 (policies as Contributors); #170 (`generateReadme`) explicitly out of scope |
| **Merge risk** | Hot zones: `convert()`, `generateHttpRoute()` — P2 should merge to `main` ASAP |
