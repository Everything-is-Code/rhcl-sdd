## Context

See `typed-yaml-infra/design.md`. Fabric8 Istio model added in phase 0 provides `ServiceEntryBuilder`, `DestinationRuleBuilder`, `EnvoyFilterBuilder`, `AuthorizationPolicyBuilder`, `TelemetryBuilder` from one artifact.

## Goals / Non-Goals

**Goals:**
- Replace `.formatted()` in all 9 Istio generators with Fabric8 typed model construction + `ManifestSerializer`.
- Keep Lua script bodies as string values inside typed `EnvoyFilter` fields.

**Non-Goals:**
- Any change to Lua generation logic (`escapeLua`, script content).
- Core K8s (phase 1), Kuadrant CRDs (phase 3), HTTPRoute (phase 4).

## Decisions

**1. `EnvoyFilterBuilder`'s `configPatches` uses nested `Map<String,Object>` for dynamic patch content.**
Rationale: Envoy's config patch schema is itself dynamic — Fabric8's model uses loosely-typed fields (`AnyType`-like). Plain map nesting fits naturally within the otherwise-typed builder. Still eliminates indentation/escaping risk via Jackson serialization.

**2. Istio `AuthorizationPolicy` is kept fully separate from Kuadrant `AuthPolicy` (phase 3).**
Rationale: unrelated CRDs (`security.istio.io/v1` vs `kuadrant.io/v1`) in separate generator classes — no shared code, no risk of conflation.

## Risks / Trade-offs

- **[Risk]** Fabric8 Istio model version may lag cluster's actual CRD schema. → **Mitigation**: use builder's `additionalProperties` escape hatch for any field not yet exposed; flag in PR for future version tracking.
- **[Risk]** `EnvoyFilterBuilder`'s loosely-typed `configPatches` reduces compile-time safety. → **Mitigation**: accepted — still eliminates escaping/indentation risk vs string templates; documented as expected.
