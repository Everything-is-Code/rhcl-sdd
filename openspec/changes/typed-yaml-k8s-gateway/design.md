## Context

See `proposal.md` and `typed-yaml-infra/design.md` for the `ManifestSerializer` facade and Jackson YAML configuration. This phase migrates the simplest tier: vanilla K8s resources with full Fabric8 model coverage.

## Goals / Non-Goals

**Goals:**
- Replace `.formatted()` in 3 generators + 1 builder + 5 contributors with Fabric8 typed model construction.
- All serialization through `ManifestSerializer.toYaml(fabric8Object)`.
- Structural test assertions via `YamlAssertions`.

**Non-Goals:**
- Istio resources (phase 2), Kuadrant CRDs (phase 3), HTTPRoute (phase 4).
- Changing Secret `stringData`/`data` semantics — preserve each contributor's current choice exactly.

## Decisions

**1. Our `SecretBuilder` (the contributor accumulator in `service/generator/contributor/`) wraps a Fabric8 `io.fabric8.kubernetes.api.model.SecretBuilder` internally, avoiding naming collision via fully-qualified import.**
Rationale: keeps the CDI contributor pattern from #40 unchanged — contributors still call our builder's methods, which delegate to the Fabric8 builder internally.

**2. All serialization goes through injected `ManifestSerializer`, not direct `Serialization.asYaml()` calls.**
Rationale: centralizes Jackson YAML configuration (field order, null handling) so all generators produce consistent output. Also makes testing easier (mock the serializer in unit tests if needed).

**3. Labels (`migrated-from: 3scale`, `app: <name>`) added via Fabric8 `.addToLabels(...)` fluent calls.**
Rationale: consistent with `ConversionYamlSupport` label-stripping conventions from #40.

## Risks / Trade-offs

- **[Risk]** Fabric8 `Serialization.asYaml()` field order may not exactly match current hand-written template order. → **Mitigation**: run full `ConversionServiceTest` before/after; any diff is either semantically equivalent (acceptable) or flagged for review.
- **[Risk]** `SecretBuilder` naming collision between our class and Fabric8's. → **Mitigation**: use fully-qualified import for the Fabric8 one inside our class; contributors never see the collision (they import only our builder).
