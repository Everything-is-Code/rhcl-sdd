## Context

See `proposal.md` for motivation. Current state:

- `ConversionService.java` — **3223 lines**, 6 `convert()` overloads, 13 `generate*()` methods, 12 `find*Policy()` helpers.
- `ConversionServiceTest` — regression oracle with whitespace-sensitive YAML assertions (~100 cases); today uses `new ConversionService()` (no CDI).
- `ArchitectureTest` — validates `service..` layering; will need extension for `generator/` and `contributor/` sub-packages.
- `dto/ConversionOptions.java` — already exists; main `convert()` overload accepts it.
- Target architecture documented in `docs/conversion-architecture.md` and enriched on GitHub #40.

Constraints: behavior-preserving refactor; no REST/API changes; no frontend work; `generateReadme(...)` refactor stays in #170; Windows CRLF may fail YAML tests locally — trust CI Linux.

## Goals / Non-Goals

**Goals:**

- Implement two-level architecture: **ResourceGenerator + Registry** (per output file) and **Contributor + Builder** (within HTTPRoute, AuthPolicy, Secret).
- Extract orchestration helpers (`BackendResolver`, `ConversionContext`), `PolicyFinder`, `StringUtils.toKebabCase()`.
- Reduce `ConversionService` to orchestrator (~100–150 lines) delegating to `ResourceGeneratorRegistry`.
- Phased delivery (P1–P6) with green tests at each phase.
- Enable #149 policy work to add Contributors without touching orchestrator.
- Enable #169 parallel bulk convert via stateless generators.

**Non-Goals:**

- Changing generated YAML semantics or adding new policy conversions (#149).
- Refactoring `generateReadme(...)` positional args (#170).
- Implementing bulk-convert parallelism (#169).
- Replacing `String.formatted()` with SnakeYAML/Qute templating.
- Frontend changes.
- `from-3scale-to-connectivity-link` adapter SDD (phase 2).

## Output file inventory

Each row maps to one `ResourceGenerator` (or conditional generator via `applies()`). Contributors aggregate **within** the parent file where noted.

| Output key | Generator | Always / conditional | Notes |
|------------|-----------|----------------------|-------|
| `gateway.yaml` | `GatewayGenerator` | Always | Listener hostnames when DNS opts enabled |
| `httproute.yaml` | `HttpRouteGenerator` | Always | Uses **HttpRoute contributors** (see P4) |
| `policy.yaml` | `AuthPolicyGenerator` | Always | Uses **AuthPolicy contributors** (see P5) |
| `secret.yaml` | `SecretGenerator` | Always | Uses **Secret contributors** (see P5) |
| `configmap.yaml` | `ConfigMapGenerator` | Always | Backend routing metadata |
| `apiproduct.yaml` | `ApiProductGenerator` | Always | Kuadrant APIProduct |
| `apikey.yaml` | `ApiKeyGenerator` | When `authentication.type == apiKey` | |
| `serviceentry.yaml` | `ServiceEntryGenerator` | When external backends exist | Multi-doc `---` join |
| `destinationrule.yaml` | `DestinationRuleGenerator` | When external backends exist | Multi-doc `---` join |
| `telemetry.yaml` | `TelemetryGenerator` | When logging policy present | |
| `envoyfilter-logging.yaml` | `LoggingEnvoyFilterGenerator` | When logging + non-empty `json_object_config` | |
| `envoyfilter-url-rewriting.yaml` | `UrlRewritingEnvoyFilterGenerator` | When url_rewriting commands present | Lua filter |
| `envoyfilter-content-limits.yaml` | `ContentLimitsEnvoyFilterGenerator` | When content_limits request bytes > 0 | |
| `envoyfilter-retry.yaml` | `RetryEnvoyFilterGenerator` | When `!retriesSupported` and retry policy set | Fallback when HTTPRoute retry unsupported |
| `authorizationpolicy.yaml` | `AuthorizationPolicyGenerator` | When ip_check + `ipCheckMode == authorizationPolicy` | Separate from `policy.yaml` |
| `ratelimitpolicy.yaml` | `RateLimitPolicyGenerator` | When edge_limiting yields YAML | |
| `tlspolicy.yaml` | `TlsPolicyGenerator` | When `includeTlsPolicy` | Opt-in |
| `dnspolicy.yaml` | `DnsPolicyGenerator` | When `emitDnsPolicy(opts)` | Opt-in |
| `README.md` | `ReadmeGenerator` | Always | Passthrough to `generateReadme` until #170 |

## Decisions

### D1 — CDI `Instance<ResourceGenerator>` for registry

**Choice:** `ResourceGeneratorRegistry` injects `Instance<ResourceGenerator>` and selects by output file key. Each generator implements `applies(ConversionContext)` for conditional files (mirrors current `if` branches in `convert()`); orchestrator builds context once and registry returns all generators where `applies()` is true.

**Alternatives:** Manual map in `@Produces` method; static registry; keep conditionals in orchestrator. CDI + `applies()` keeps orchestrator thin and conditional logic testable per generator.

### D2 — Contributor discovery via CDI `Instance<Contributor>` per builder type

**Choice:** `HttpRouteGenerator` injects `Instance<HttpRouteContributor>`; same pattern for AuthPolicy and Secret.

**Alternatives:** Hardcoded contributor list in generator; single mega-interface. Per-type `Instance` keeps contributors isolated and testable.

### D3 — Passthrough generators in P2 before logic extraction

**Choice:** P2 wraps existing `generate*()` methods in generator classes without moving logic yet; orchestrator loops registry.

**Alternatives:** Big-bang extraction. Passthrough minimizes risk, lands registry spine early, reduces merge conflict window for P3–P5.

### D4 — `dto.ConversionOptions` (existing) — audit, do not duplicate

**Choice:** Keep `dto/ConversionOptions.java` as the options carrier (mutable class with flags: `corsNative`, `retriesSupported`, `includeTlsPolicy`, `ipCheckMode`, DNS/TLS fields). Audit overloads; extend DTO if any flag still travels as a loose parameter. `ConversionContext` holds per-run derived state (resolved backends, kebab name, etc.) separate from user-facing options.

**Alternatives:** New record in `service/` — rejected to avoid duplicate type and controller binding churn.

### D5 — `PolicyFinder.findEnabled(ApiService, String policyName)` replaces 12 finders

**Choice:** `@ApplicationScoped` `PolicyFinder` with generic lookup over policy chain by name.

**Alternatives:** Static util; per-policy finders in generators. Central bean reduces duplication; generators/contributors inject it.

### D6 — README stays as passthrough generator delegating to current method

**Choice:** `ReadmeGenerator` calls existing `generateReadme(...)` until #170.

**Rationale:** Avoid merge conflict with #170; do not add positional parameters per project rule.

### D7 — Orchestration layer (`service/conversion/`)

**Choice:** Extract per-run orchestration from `ConversionService` into dedicated types before registry loop:

| Class | Responsibility |
|-------|----------------|
| `ResolvedBackend` | Backend resolution result (internal/external, host, port, TLS, names) |
| `BackendResolver` | `resolveBackends`, `resolveOne`; URL parsing via `UrlUtils` |
| `ConversionContext` | Built once per `convert()`: kebab service name, namespace, `backendUrl`, `ConversionOptions`, resolved backends, primary type/host, `overrideIgnored` flag |
| `ConversionYamlSupport` | `parseJsonObjectConfig`, `stripMigratedFromLabel` |
| `PolicyConfigSupport` | `resolveRetryAttempts`, `resolveContentLimitBytes`, shared policy config parsing |

`ConversionService.convert()` becomes: build context → call registry → assemble `LinkedHashMap` (order preserved).

**Alternatives:** Keep all helpers private in `ConversionService` — rejected; blocks reaching ~150-line orchestrator.

### D8 — Package layout

```
dto/
  ConversionOptions.java       (existing — flags for convert API)
service/
  ConversionService.java       (~100–150 lines orchestrator)
  PolicyFinder.java
service/conversion/
  ResolvedBackend.java
  BackendResolver.java
  ConversionContext.java
  ConversionYamlSupport.java
  PolicyConfigSupport.java
service/generator/
  ResourceGenerator.java       (interface: outputKey(), applies(), generate())
  ResourceGeneratorRegistry.java
  GatewayGenerator, HttpRouteGenerator, AuthPolicyGenerator, SecretGenerator, ...
  LoggingEnvoyFilterGenerator, UrlRewritingEnvoyFilterGenerator, ...
  AuthorizationPolicyGenerator, TlsPolicyGenerator, DnsPolicyGenerator, ...
service/generator/contributor/
  HttpRouteContributor, HttpRouteBuilder, CorsContributor, ...
  AuthPolicyContributor, AuthPolicyBuilder, ...
  SecretContributor, SecretBuilder, ...
util/
  StringUtils.java             (toKebabCase)
  UrlUtils.java                (extractHostname, extractPort, ...)
```

Layering: `conversion/` → `model`, `util`, `dto`; `generator/` → `conversion`, `service`, `model`, `util`; `contributor/` → `conversion`, `model`, `util`; none → `controller`, `client`. Contributors MUST NOT depend on `ConversionService` (ArchUnit).

## Test strategy

Three layers — see `tasks.md` for concrete test class names.

### Layer 1 — Regression oracle (`ConversionServiceTest`)

- **Role:** Behavior preservation; ~100 YAML string assertions.
- **Today:** Plain JUnit, `service = new ConversionService()` — bypasses CDI.
- **After P2:** Two options (pick one in P2 PR, document in PR):
  - **A (preferred):** Migrate to `@QuarkusTest` + `@Inject ConversionService` so orchestrator uses real registry wiring.
  - **B:** Keep plain tests but add `ConversionServiceTestSupport` that manually constructs registry + generators for parity with production wiring.
- **CRLF:** Trust CI Linux if local Windows fails on whitespace-only diffs.

### Layer 2 — Unit tests (per class)

- `PolicyFinderTest`, `StringUtilsTest`, `UrlUtilsTest`, `BackendResolverTest`, `ConversionContextTest`
- Per-generator tests with minimal `ApiService` fixtures (no full Quarkus boot where unnecessary)
- Per-contributor tests applying builder fragments in isolation

### Layer 3 — CDI discovery tests (`@QuarkusTest`)

Prove #149 extensibility: new bean on classpath → output changes **without** editing orchestrator or parent generator.

| Test class | Injects | Proves |
|------------|---------|--------|
| `ResourceGeneratorRegistryDiscoveryTest` | `ResourceGeneratorRegistry` | New `ResourceGenerator` bean discovered |
| `HttpRouteContributorDiscoveryTest` | `HttpRouteGenerator` | New `HttpRouteContributor` bean applied |
| `AuthPolicyContributorDiscoveryTest` | `AuthPolicyGenerator` | New `AuthPolicyContributor` bean applied |
| `SecretContributorDiscoveryTest` | `SecretGenerator` | New `SecretContributor` bean applied |

Dummy beans live in `src/test/java/.../generator/discovery/` with YAML marker strings; each suite has positive + negative case.

### Layer 4 — Architecture & integration

- `ArchitectureTest` — layering for `conversion`, `generator`, `contributor`; contributors ≠ `ConversionService`
- `ConversionServiceConcurrencyTest` — parallel `convert()` calls, no shared mutable state (P6)
- REST smoke — convert/preview endpoints unchanged (`docs/api-spec.yml`)

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| Merge conflicts during P3–P5 with in-flight #149 PRs | Merge P2 to `main` ASAP; coordinate: new policies must land as Contributors |
| `ConversionServiceTest` CRLF failures on Windows | Trust CI Linux; document in tasks verification |
| CDI migration breaks plain `new ConversionService()` tests | Explicit test strategy (Layer 1 A or B) in P2 |
| ArchUnit gaps allow layering violations | Add rules in P2; discovery tests + contributor ArchUnit rule |
| Passthrough P2 feels like dead code | Short-lived; P3 removes duplication quickly |
| Contributor ordering affects YAML fragment order | `@Priority` or explicit sort matching pre-refactor output |
| Large PR if phases skipped | Enforce phased tasks; one PR per phase where possible |

## Migration Plan

| Phase | Deliverable | Merge target |
|-------|-------------|--------------|
| **P1 — Spine** | Audit `dto.ConversionOptions`; `PolicyFinder`; `StringUtils`; replace 12 `find*Policy` | Small PR |
| **P2a — Orchestration** | `service/conversion/` (`BackendResolver`, `ConversionContext`, `ConversionYamlSupport`, `PolicyConfigSupport`, `UrlUtils`) | With P2b |
| **P2b — Registry** | `ResourceGenerator` + registry + **passthrough for all output files** (see inventory table); orchestrator loop; discovery test for registry | **Merge to main early** |
| **P3 — Simple generators** | Move logic into generators without contributors: Gateway, ServiceEntry, DestinationRule, Telemetry, all four EnvoyFilters, ConfigMap, RateLimitPolicy, ApiProduct, ApiKey, AuthorizationPolicy, TlsPolicy, DnsPolicy | 1–2 PRs |
| **P4 — HTTPRoute contributors** | `HttpRouteBuilder` + contributors: MappingRules, MultiBackend, HeaderMod, Cors, UrlRewrite (HTTPRoute fragments), Retry (`retriesSupported`), Timeouts, HttpRouteAnnotations; discovery test | 1 PR |
| **P5 — AuthPolicy contributors** | `AuthPolicyBuilder` + contributors: Anonymous, Oauth2Introspection, AuthCaching, IpCheckOpa, JwtClaimCheck, KeycloakRoleCheck; `AuthPolicyYamlMerger`; discovery test | 1 PR |
| **P5 — Secret contributors** | `SecretBuilder` + contributors: AnonymousSecret, TokenIntrospectionSecret, AppIdKeySecret; discovery test | 1 PR (or with P5 Auth) |
| **P6 — Cleanup** | Delete orphan methods; `ConversionServiceConcurrencyTest`; final ArchUnit; README passthrough only | 1 PR |

### Contributor checklist (P4 / P5)

**HttpRoute (`httproute.yaml`):**

| Contributor | Source logic today |
|-------------|-------------------|
| `MappingRulesContributor` | Path rules, `buildRuleFiltersBlock` |
| `MultiBackendContributor` | Per-backend rules, shared filters |
| `HeaderModContributor` | `buildHeaderModificationFilters` |
| `CorsContributor` | `buildCorsFilters`, `corsNative` flag |
| `UrlRewriteContributor` | HTTPRoute-native rewrite fragments (Envoy path stays in `UrlRewritingEnvoyFilterGenerator`) |
| `RetryContributor` | `buildRetryBlock` when `retriesSupported` |
| `TimeoutsContributor` | `buildTimeoutsBlock` |
| `HttpRouteAnnotationsContributor` | `buildHttpRouteAnnotations`, `buildUpstreamAnnotations` |

**AuthPolicy (`policy.yaml`):**

| Contributor | Source logic today |
|-------------|-------------------|
| `AnonymousContributor` | `generateAnonymousAuthPolicy`, `anonymousTarget` |
| `Oauth2IntrospectionContributor` | `generateOauth2IntrospectionAuthPolicy` |
| `AuthCachingContributor` | `buildAuthCacheBlock` (with introspection) |
| `IpCheckOpaContributor` | `appendIpCheckOpaIfNeeded`, `ipCheckMode == authPolicyOpa` |
| `JwtClaimCheckContributor` | `appendJwtClaimCheckAuthorization` |
| `KeycloakRoleCheckContributor` | `appendKeycloakRoleCheckAuthorization` |

Note: `authorizationpolicy.yaml` is a **separate generator** (`AuthorizationPolicyGenerator`), not an AuthPolicy contributor — IpCheck in `authorizationPolicy` mode.

**Secret (`secret.yaml`):**

| Contributor | Source logic today |
|-------------|-------------------|
| `AnonymousSecretContributor` | Anonymous policy credentials |
| `TokenIntrospectionSecretContributor` | `generateTokenIntrospectionSecret` |
| `AppIdKeySecretContributor` | `generateAppIdKeySecret`, api_key auth |

Rollback: each phase is independently revertible; P2 passthrough preserves identical behavior.

## Open Questions

| Question | Default if unresolved |
|----------|----------------------|
| `ConversionServiceTest` CDI migration (Layer 1 A vs B) | **A** — `@QuarkusTest` in P2 |
| Contributor ordering mechanism | `@Priority` on contributors, explicit sort in generator matching current YAML order |
| Generator bean scope | `@ApplicationScoped`, stateless (required for #169 parallelism) |

None blocking — phasing and package layout confirmed on #40 enrich-us comment.
