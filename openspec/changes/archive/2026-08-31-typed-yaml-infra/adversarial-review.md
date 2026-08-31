# Adversarial review — typed-yaml-infra

**Scope:** OpenSpec change `typed-yaml-infra` / [#262](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/262) Phase 0  
**Sources:** `openspec/changes/typed-yaml-infra/{proposal,design,tasks}.md`, `../migration-toolkit-rhcl/` branch `feature/262-typed-yaml-infra` (uncommitted)  
**Date:** 2026-08-31  
**Reviewer:** Independent adversarial pass (post code-review fixes)

## Adversarial review

### Spec alignment

| Criterion (proposal / tasks) | Adversarial read |
|------------------------------|------------------|
| No generator/contributor production edits | **Holds** — diff is additive only under `model/kuadrant/`, `service/conversion/`, tests, `pom.xml`, `ArchitectureTest` |
| `model/kuadrant/` records for 6 CRD types | **Holds** — types exist; structural tests present |
| `ManifestSerializer` unified path | **Partial** — records use Jackson; Fabric8 `HasMetadata` still uses `Serialization.asYaml()` (dual formatter by design, documented) |
| Golden validation vs `.formatted()` templates | **Holds** — `KuadrantManifestEquivalenceTest` covers all 6 Kuadrant manifests (TLS, DNS with/without provider, APIProduct including quoted description, APIKey, AuthPolicy empty + JWT, RateLimit edge limiting) |
| ArchUnit shrinking allowlist | **Holds** — generator (35) + conversion (5) allowlists; `ReadmeSupport` excluded |
| Codecov / unhappy-path tests | **Holds** — null/empty/special-char cases; full suite green |
| `skip_specs: true` / no user behavior change | **Holds** — nothing wired into `ConversionService` yet |

### Findings (resolved)

| Severity | Area | Finding | Resolution |
|----------|------|---------|------------|
| Major | Equivalence coverage | AuthPolicy + RateLimitPolicy had no generator↔record equivalence | Added `authPolicy_emptyAuthentication_*`, `authPolicy_jwtAuthentication_*`, `rateLimitPolicy_edgeLimiting_*` in `KuadrantManifestEquivalenceTest` |
| Major | YAML comments | RateLimit `# WARNING` comments not round-trippable | Documented in `typed-yaml-infra/design.md` decision #7 and `typed-yaml-kuadrant/design.md` decision #5; structural equivalence ignores comments (parser) |
| Minor | Equivalence gaps | DNS without `providerRefs` untested | Added `dnsPolicy_withoutProvider_recordMatchesGeneratorOutput` |
| Minor | Equivalence gaps | ApiProduct quoted description untested in golden equiv | Added `apiProduct_quotedDescription_recordMatchesGeneratorOutput` |
| Minor | ArchUnit blind spot | `service.conversion` `.formatted()` uncaught | Second ArchUnit rule + `FORMATTED_YAML_CONVERSION_ALLOWLIST` (5 classes) |
| Minor | Artifact drift | `tasks.md` istio artifact name; stale test count | `tasks.md` updated to `istio-model`; `verify-report.md` updated |
| Minor | Constants | `FABRIC8_VERSION` duplicated pom | Removed from `YamlSerializationConfig` |
| Process | Uncommitted | No PR yet | Still pending user commit/PR request |

### Attack scenarios considered (negative paths)

1. **Null manifest** → `IllegalArgumentException` — covered (`ManifestSerializerTest`).
2. **Special YAML chars in metadata** — covered; parses back correctly.
3. **Empty AuthPolicy sections** — `authentication: {}` with omitted `authorization` matches `EmptyAuthenticationContributor` output.
4. **New generator adds `.formatted()` without allowlist update** — ArchUnit should fail — good guardrail.
5. **Conversion output regression** — **not possible yet** (generators unchanged); risk is latent for phase 3.
6. **Secrets in YAML** — no new logging; records don't alter secret handling.

### Verdict

**PASS**

Phase 0 infrastructure is complete with full Kuadrant golden equivalence coverage and documented RateLimit comment strategy for phase 3. Remaining process item: commit + PR.

### Before archive

- [ ] Commit + PR `feature/262-typed-yaml-infra` (process)
- [x] AuthPolicy + RateLimit golden equivalence tests
- [x] RateLimit `# WARNING` comment strategy documented
- [x] DNS without-provider + ApiProduct quoted-description equivalence tests
- [x] `tasks.md` istio artifact wording updated
