# Review report — typed-yaml-infra

**Change:** `typed-yaml-infra`  
**Issue:** [#262](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/262)  
**Branch:** `feature/262-typed-yaml-infra` (uncommitted)  
**Reviewer:** AI (Cursor)  
**Date:** 2026-08-31

## AI Code Review

_Disclaimer: AI-generated review. Human maintainer review required._

**Change:** `typed-yaml-infra` | **Issue:** #262

### Major

_None._

### Moderate

1. **No golden round-trip against live generators yet** — Manifest tests assert structural fields (e.g. `TlsPolicyManifestTest`) but do not compare `ManifestSerializer.toYaml(record)` to the current `*Generator.generate()` string output. Phase 0 is infra-only, but phase 3 (`typed-yaml-kuadrant`) should add at least one fixture-based equivalence test per manifest type before swapping generators, or risk silent YAML drift (field order, quoting, omitted empty sections).

2. **`YamlAssertions` vs `ManifestSerializer` YAML config divergence** — `YamlAssertions` uses a default `YAMLFactory`, while `ManifestSerializer` disables doc markers and enables `MINIMIZE_QUOTES`. Structural parse tests may pass even when serialized strings would not byte-match generator templates. Consider sharing `ManifestSerializer.createYamlMapper()` (or a test-scoped copy) in `YamlAssertions` before migration phases rely on it for regression checks.

3. **`istio-model` version pinned explicitly (`7.8.0`)** — Matches current `kubernetes-client` tree today, but bypasses Quarkus BOM alignment. If the platform bumps Fabric8 independently, this dep could drift. Prefer a `${fabric8.version}` property sourced from the same BOM line or a comment linking to the kubernetes-client version in `dependency:tree`.

### Minor

1. **`CacheConfig` / `ResponseConfig` shapes untested** — Records exist for auth-cache and anonymous `response.success.headers` paths, but no dedicated serialization tests. These are the highest-risk AuthPolicy areas in phase 3 (#53 CORS/indentation history).

2. **`RateLimitPolicyManifest` omits `Counter` record from tasks** — `tasks.md` mentions `Limit`/`Rate`/`Counter`; implementation has `Rate` + `LimitDefinition` only. Acceptable if connection-limit semantics map to `Rate`, but document in phase 3 design or add the type when migrating `RateLimitSupport` comment blocks.

3. **`AuthPolicyRules` null vs empty map semantics** — `@JsonInclude(NON_NULL)` omits null `authentication`/`authorization` keys (see `authorizationRule_serializesPatternMatchingBlock` passing `null` authentication). Current generators always emit both section keys when present. Phase 3 builders should use `Map.of()` not `null` to preserve `{}` vs omitted distinction.

4. **Smoke tests live under `model.kuadrant`** — `GatewayApiModelSmokeTest` / `IstioModelSmokeTest` validate dependencies, not Kuadrant models. Consider `service.conversion` or a `support` test package for clarity.

### Nit

1. **ArchUnit allowlist is 35 classes, not 40** — Matches actual `service.generator..` `.formatted()` usage (conversion support correctly out of scope per tasks). Update proposal/tasks wording to avoid confusion.

2. **ArchUnit rule matches `String.formatted()` only** — Does not catch `String.format()` / `StringBuilder` YAML assembly (e.g. `AnonymousContributor`, `JwtClaimCheckSupport.buildNamedRule`). Acknowledged in design; no action for phase 0.

3. **`AuthenticationRule` / `AuthorizationRule` null `value` map** — No guard; `new AuthenticationRule(null)` would likely NPE at serialize time. Low risk while only tests construct instances.

## Positive notes

- Clean phase-0 boundary: no generator/contributor production edits; behavior-preserving scope respected.
- Good unhappy-path coverage for `ManifestMeta`, `ManifestSerializer`, and `YamlAssertions`.
- ArchUnit shrinking allowlist with documented workflow is the right guardrail for phases 1–4.
- Layering intact: records in `model.kuadrant`, serializer in `service.conversion`, test helper in `support`.
- Full suite green locally (`mvn test` 986/986).

## Verdict

**Approve** — review findings addressed; safe to merge as phase-0 infrastructure.

## Resolution log (2026-08-31)

| Finding | Resolution |
|---------|------------|
| Moderate: no golden round-trip | Added `KuadrantManifestEquivalenceTest` (TLS, DNS, APIProduct, APIKey) with `MapEquivalenceSupport` |
| Moderate: YamlAssertions config drift | Extracted `YamlSerializationConfig.createYamlMapper()` shared by serializer + assertions |
| Moderate: istio-model version pin | `${fabric8.version}` property + comment aligned with kubernetes-client |
| Minor: CacheConfig/ResponseConfig untested | Added `cacheConfig_serializesInsideAuthenticationRule` and `responseConfig_serializesAnonymousSuccessHeaders` |
| Minor: Counter record | Documented on `LimitDefinition` (conn limits → `Rate` with `1s` window) |
| Minor: null vs empty AuthPolicy maps | Compact constructors default null maps to `Map.of()` |
| Minor: smoke test package | Moved to `service.conversion` |
| Nit: allowlist count 40 vs 35 | Updated `proposal.md` to 35 |
