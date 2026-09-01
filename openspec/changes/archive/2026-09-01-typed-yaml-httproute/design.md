## Context

See `typed-yaml-infra/design.md` and `typed-yaml-k8s-gateway/design.md` decision #1 (naming collision pattern for our builder wrapping Fabric8's). Fabric8 Gateway API model provides `HTTPRouteBuilder`, `HTTPRouteRule`, `HTTPRouteFilter`, `HTTPBackendRef` — full coverage for this resource type.

## Goals / Non-Goals

**Goals:**
- Replace `.formatted()` in all 12 listed classes with Fabric8 Gateway API typed model construction + `ManifestSerializer`.
- Preserve `@Priority` contributor ordering from #40 exactly.
- Empty the ArchUnit allowlist after this phase — hard guard against new `.formatted()` YAML from this point.

**Non-Goals:**
- Changing route-matching semantics, filter selection logic, or backend weighting.
- Other phases.

## Decisions

**1. Our `HttpRouteBuilder` wraps a Fabric8 `HTTPRouteBuilder` internally — same pattern as `SecretBuilder` in phase 1.**
Rationale: contributors' CDI injection points and method-call shape don't change; only what they construct internally changes.

**2. Filter-producing contributors each append a typed `HTTPRouteFilter` (discriminated union: `ResponseHeaderModifier`, `RequestHeaderModifier`, `URLRewrite`, etc.) to the rule's filter list.**
Rationale: Gateway API's filter model is well-defined in Fabric8 — using it gives compile-time structure for exactly the kind of filter-type/field mismatch that caused bugs in the string-template era.

**3. `HttpRouteAnnotationsContributor` writes to `metadata.annotations` (plain map) — no typed model benefit.**
Rationale: annotations are inherently untyped in K8s. Migration is mechanical: map entry instead of formatted string.

**4. Support classes (`HttpRouteSupport`, `RoutingSupport`, `UpstreamSupport`) return typed Fabric8 objects instead of YAML string fragments.**
Rationale: support classes are called by contributors to build sub-structures (filters, matches, backendRefs) — returning typed objects lets contributors compose them directly into the `HTTPRouteRule` builder without string concatenation.

## Risks / Trade-offs

- **[Risk]** Highest regression risk: most contributors on one shared output file, ~30 `ConversionServiceTest` assertions. → **Mitigation**: rebase onto latest `main` before starting; verify `@Priority` order unchanged as explicit checklist item.
- **[Risk]** Gateway API's `HTTPRouteFilter` discriminated-union may not expose every filter variant equally. → **Mitigation**: fall back to builder's generic extension points if needed; never revert to string formatting.
