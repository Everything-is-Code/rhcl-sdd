# Tasks — typed-yaml-k8s-gateway

GitHub: [#262](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/262) — Phase 1

## Prerequisites

- [ ] 0.1 Confirm `typed-yaml-infra` (phase 0) is merged — `ManifestSerializer`, `YamlAssertions` available
- [ ] 0.2 Create feature branch: `feature/262-typed-yaml-k8s-gateway` from current `main`
- [ ] 0.3 Confirm CI green on `main`

## 1. Gateway

- [ ] 1.1 Rewrite `GatewayGenerator.java` to build `GatewayBuilder` (name, namespace, labels, `gatewayClassName`, listeners with DNS-derived hostnames), serialize via `ManifestSerializer.toYaml(...)` — verify `GatewayGeneratorTest` passes
- [ ] 1.2 Migrate `GatewayGeneratorTest` to `YamlAssertions.parse(...)` structural checks — verify passes
- [ ] 1.3 Remove `GatewayGenerator` from ArchUnit allowlist — verify `ArchitectureTest` passes

## 2. ConfigMap

- [ ] 2.1 Rewrite `ConfigMapGenerator.java` via `ConfigMapBuilder` — verify `ConfigMapGeneratorTest` passes
- [ ] 2.2 Migrate test to structural assertions — verify passes
- [ ] 2.3 Remove from ArchUnit allowlist — verify `ArchitectureTest` passes

## 3. Secret (shared builder + generator)

- [ ] 3.1 Rewrite `SecretBuilder.java` to wrap a Fabric8 `SecretBuilder` internally, exposing `addStringData(key, value)` / `addLabel(...)` — verify `SecretBuilderTest` passes
- [ ] 3.2 Rewrite `SecretGenerator.java` to serialize via `ManifestSerializer.toYaml(...)` — verify `SecretGeneratorTest` passes
- [ ] 3.3 Confirm `SecretContributorDiscoveryTest` still passes unmodified

## 4. Secret contributors

- [ ] 4.1 Migrate `AnonymousSecretContributor` — verify `AnonymousSecretContributorTest` passes
- [ ] 4.2 Migrate `ApiKeySecretContributor` — verify test passes
- [ ] 4.3 Migrate `AppIdKeySecretContributor` (2 call sites) — verify test passes
- [ ] 4.4 Migrate `DefaultCredentialsSecretContributor` — verify test passes
- [ ] 4.5 Migrate `TokenIntrospectionSecretContributor` — verify test passes
- [ ] 4.6 Remove all secret classes from ArchUnit allowlist — verify `ArchitectureTest` passes

## 5. Edge-case / unhappy-path tests

- [ ] 5.1 Test `GatewayGenerator` with a service that has no custom domain (null/empty `systemName`) — verify produces a valid Gateway with default hostname or throws descriptively
- [ ] 5.2 Test `GatewayGenerator` with a service name containing special DNS-invalid chars — verify they are sanitized or rejected with clear error
- [ ] 5.3 Test `ConfigMapGenerator` with empty data map — verify produces a valid ConfigMap with `data: {}`
- [ ] 5.4 Test `SecretBuilder` with duplicate `addStringData(key, value)` calls for the same key — verify last-write-wins behavior (not duplicated keys in YAML)
- [ ] 5.5 Test `SecretBuilder` with no contributors adding data — verify produces a valid Secret with empty `stringData`
- [ ] 5.6 Test secret contributors with null/empty credential values (e.g. `apiKey = ""`) — verify behavior is predictable (empty string in output, not crash)

## 6. Regression

- [ ] 6.1 Run full `ConversionServiceTest` — update any gateway/configmap/secret string assertions that differ only in field order (document in PR) — verify CI Linux green
- [ ] 6.2 Run `ConversionServiceConcurrencyTest` — verify passes

## Verification

```bash
cd ../migration-toolkit-rhcl/backend && mvn test
cd ../migration-toolkit-rhcl/backend && mvn verify
```

## Verify

- [ ] Run `/verify` — record result in `verify-report.md`
