## Context

See `typed-yaml-infra/design.md` decisions #2, #3, #5 for record design principles. The `model/kuadrant/` records created in phase 0 model only the fields generators actually emit. This phase plugs the records into the existing generator/contributor CDI architecture.

## Goals / Non-Goals

**Goals:**
- Replace all `.formatted()` in the 18 listed classes with typed record construction + `ManifestSerializer`.
- Preserve the CDI contributor pattern: contributors call typed builder methods producing typed section records, not string fragments.
- Eliminate the `ApiProductGenerator` `description.replace("\"", "'")` escaping hack — Jackson handles quoting automatically.

**Non-Goals:**
- Any change to which AuthPolicy/RateLimitPolicy fields are emitted or their semantics.
- Fabric8 Java Generator for Kuadrant CRDs (rejected in #262 enrichment).

## Decisions

**1. `AuthPolicyBuilder` exposes typed accumulator methods per AuthPolicy section: `addAuthentication(name, AuthenticationRule)`, `addAuthorization(name, AuthorizationRule)`, `setResponse(ResponseConfig)`.**
Rationale: Kuadrant's `AuthPolicy.spec.rules.{authentication,authorization,response}` structure is fixed — typed methods catch a contributor writing to the wrong section at compile time. Compare map approach: `builder.put("authentication", Map.of(...))` — typos in key names compile silently.

**2. Builder produces an immutable `AuthPolicyManifest` record via `build()` — standard Builder pattern, same as Fabric8 internally.**
Rationale: mutable accumulation during contributor phase, immutable result for serialization — clean separation, thread-safe output.

**3. `RateLimitSupport` numeric formatting moves from `%d`/`%s` to typed record fields (`int limit`, `String window`).**
Rationale: SnakeYAML/Jackson serialize `Integer` as unquoted `42` and `String` as quoted-if-needed text automatically — eliminates the class of bugs where numeric fields are accidentally quoted inconsistently.

**4. Named-rule ordering preserved via `LinkedHashMap` in builder accumulator.**
Rationale: preserves current contributor execution order exactly, avoiding the question of whether Kuadrant's admission logic is order-sensitive.

**5. RateLimit `# WARNING` comment lines are dropped when migrating to typed records.**
Rationale: Jackson cannot serialize YAML comments. `RateLimitSupport` warning text (leaky-bucket approximation, connection-limit semantics, plan-ceiling scope) remains in README output via `ReadmeSupport`; typed `RateLimitPolicyManifest` carries only structural `limits` data. If inline warnings must survive in `ratelimitpolicy.yaml`, phase 3 can prepend a custom comment block via a dedicated serializer — default is accept comment loss with README coverage.

## Risks / Trade-offs

- **[Risk]** Largest phase (18 classes) — highest chance of subtle behavior change. → **Mitigation**: consider splitting into 2 PRs (AuthPolicy sub-group, then everything else) if review load is too high. Tasks are ordered for this split.
- **[Risk]** `AuthPolicyManifest` record nesting depth (spec.rules.authentication.jwt.{issuer,audiences,...}) could make construction verbose. → **Mitigation**: nested records with sensible defaults; Builder methods handle the nesting so contributors don't see it directly.
