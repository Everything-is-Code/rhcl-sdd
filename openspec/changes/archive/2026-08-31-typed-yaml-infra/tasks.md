# Tasks — typed-yaml-infra

GitHub: [#262](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/262) — Phase 0

## Prerequisites

- [x] 0.1 Review `proposal.md` and `design.md` in this change
- [x] 0.2 Create feature branch: `feature/262-typed-yaml-infra` from current `main`
- [x] 0.3 Confirm CI green on `main` (`cd backend && mvn test`)

## 1. Dependencies

- [x] 1.1 Add `jackson-dataformat-yaml` to `backend/pom.xml` — verify `mvn -q -DskipTests dependency:tree` shows it resolved
- [x] 1.2 Add Fabric8 Istio model dependency (`io.fabric8:istio-model`, version `${fabric8.version}` from Quarkus BOM) — verify resolved, no version conflicts
- [x] 1.3 Confirm existing `quarkus-kubernetes-client` provides `io.fabric8.kubernetes.api.model.gatewayapi.v1.GatewayBuilder` and `HTTPRouteBuilder` — verify a smoke test constructs both successfully

## 2. Shared record types

- [x] 2.1 Create `model/kuadrant/ManifestMeta.java` — record with `name`, `namespace`, `labels` (Map), nullable `annotations` (Map), `@JsonInclude(NON_NULL)` — verify `ManifestMetaTest` serialization round-trip
- [x] 2.1.1 Test `ManifestMeta` with null `annotations` — verify field omitted from YAML output (not serialized as `annotations: null`)
- [x] 2.1.2 Test `ManifestMeta` with empty `labels` map — verify serializes as `labels: {}`
- [x] 2.1.3 Test `ManifestMeta` with null `name` — verify behavior (should fail fast or produce predictable output)
- [x] 2.2 Create `model/kuadrant/TargetRef.java` — record with `group`, `kind`, `name` — verify serialization test
- [x] 2.2.1 Test `TargetRef` with empty/blank `name` — verify behavior
- [x] 2.3 Create `model/kuadrant/TlsPolicyManifest.java` — record with `apiVersion`, `kind`, `metadata`, `spec` (nested `TlsPolicySpec` with `targetRef`, `issuerRef`) — verify serialization matches current `TlsPolicyGenerator` output structure by comparing against the existing text block template
- [x] 2.4 Create `model/kuadrant/DnsPolicyManifest.java` — same approach, verify against `DnsPolicyGenerator` template
- [x] 2.5 Create `model/kuadrant/ApiProductManifest.java` — verify against `ApiProductGenerator` template
- [x] 2.6 Create `model/kuadrant/ApiKeyManifest.java` — verify against `ApiKeyGenerator` template
- [x] 2.7 Create `model/kuadrant/RateLimitPolicyManifest.java` with nested `Limit`/`Rate`/`Counter` records — verify against `RateLimitSupport` output structures (5 variants: ceiling, count/window, rate, connLimit, full envelope)
- [x] 2.8 Create `model/kuadrant/AuthPolicyManifest.java` with nested section records: `AuthPolicySpec`, `AuthenticationRule`, `AuthorizationRule`, `ResponseConfig`, `TargetRef`, `CacheConfig` — verify against `AuthPolicyGenerator`/`AuthPolicyBuilder` output structures (the most complex record: authenticate each nested level by reading the actual `.formatted()` templates)
- [x] 2.9 Test each manifest record with minimal required fields only — verify YAML output omits optional fields cleanly
- [x] 2.10 Test `RateLimitPolicyManifest` with zero rate / zero limit — verify serializes as `0` (not omitted)
- [x] 2.11 Test `AuthPolicyManifest` with empty authentication/authorization maps — verify serializes as `{}` (not omitted)
- [x] 2.12 Run all record serialization tests — verify `mvn test -Dtest=*ManifestTest,ManifestMetaTest,TargetRefTest` passes

## 3. ManifestSerializer

- [x] 3.1 Create `service/conversion/ManifestSerializer.java` — `@ApplicationScoped` CDI bean wrapping a pre-configured `ObjectMapper` (YAML factory, `NON_NULL`, `MINIMIZE_QUOTES`, no doc start marker, block style); `toYaml(Object)` method dispatches to `Serialization.asYaml()` for Fabric8 `HasMetadata` instances, Jackson `writeValueAsString()` for records — verify `ManifestSerializerTest` passes
- [x] 3.2 Test serializer with one Fabric8 object (`SecretBuilder`) and one custom record (`TlsPolicyManifest`) — verify both produce valid YAML with expected field order and no null-field noise
- [x] 3.3 Test `toYaml(null)` — verify throws `NullPointerException` or `IllegalArgumentException` with descriptive message (not silent empty string)
- [x] 3.4 Test serializer with a record containing special YAML chars in string fields (colons, `#`, `{`, multiline values) — verify output is properly quoted/escaped
- [x] 3.5 Test serializer with a record that has all-null optional fields — verify output contains only non-null fields (no `field: null` lines)

## 4. YamlAssertions test helper

- [x] 4.1 Create `src/test/java/.../support/YamlAssertions.java` with `assertValidYaml(String)` and `parse(String) : Map<String,Object>` — verify `YamlAssertionsTest` covers valid/invalid YAML parsing and clear error messages on malformed input
- [x] 4.2 Test `assertValidYaml` with malformed YAML (missing closing bracket, bad indentation, tabs mixed with spaces) — verify throws `AssertionError` with actionable message showing the parse error
- [x] 4.3 Test `parse(null)` and `parse("")` — verify throws with clear message (not NPE without context)
- [x] 4.4 Test `parse` with valid YAML that is not a map (e.g. a bare scalar, a list) — verify throws or returns predictable result

## 5. ArchUnit scaffold rule

- [x] 5.1 Add ArchUnit rules in `ArchitectureTest` flagging `.formatted()` on text blocks in `service.generator..` and `service.conversion..` (excluding `ReadmeSupport`), with explicit allowlists — verify `mvn test -Dtest=ArchitectureTest` passes with allowlists in place
- [x] 5.2 Document the allowlist shrink process in a comment above the rule — verify comment present

## 6. Smoke test for unused dependencies

- [x] 6.1 Add a minimal test constructing one Fabric8 Istio object (e.g. `new ServiceEntryBuilder().build()`) to prove the Istio dep is in use — verify compiles and runs

## Verification

```bash
cd ../migration-toolkit-rhcl/backend && mvn test -Dtest=*ManifestTest,ManifestMetaTest,TargetRefTest,ManifestSerializerTest,YamlAssertionsTest,ArchitectureTest
cd ../migration-toolkit-rhcl/backend && mvn verify
```

## Docs

- [x] 7.1 Update `docs/sdd-backlog.md` with the 5 new typed-yaml changes

## Verify

- [x] Run `/verify` — record result in `verify-report.md`
