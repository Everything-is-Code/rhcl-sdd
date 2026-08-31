## Why

`service/generator/**` and `service/conversion/**` build Kubernetes YAML manifests via `String.formatted()` text blocks (~49 call sites across 40 files, excluding `ReadmeSupport`). Before migrating any individual generator to typed Java objects (phases 1-4 of epic #262), the shared infrastructure must exist: record types for Kuadrant CRDs, a unified serialization facade, test assertion helpers, and the necessary Fabric8/Jackson dependencies.

## What Changes

- Add `model/kuadrant/` package with Java record types modeling the 6 Kuadrant CRD resource types the tool emits: `AuthPolicyManifest`, `RateLimitPolicyManifest`, `TlsPolicyManifest`, `DnsPolicyManifest`, `ApiProductManifest`, `ApiKeyManifest`, plus shared records (`ManifestMeta`, `TargetRef`).
- Add `service/conversion/ManifestSerializer.java`: facade that serializes both Fabric8 model objects (`Serialization.asYaml(...)`) and custom records (`ObjectMapper` with `jackson-dataformat-yaml`) through a single entry point.
- Add `jackson-dataformat-yaml` dependency to `backend/pom.xml`.
- Add Fabric8 Istio model dependency to `backend/pom.xml` (needed by phase 2).
- Confirm existing `quarkus-kubernetes-client` provides Gateway API model classes (needed by phases 1/4).
- Add shared test utility `YamlAssertions` with `assertValidYaml(String)` and `parse(String)` for structural test assertions.
- Add ArchUnit rule (with allowlist of all 35 `service.generator..` classes currently using `String.formatted()`) flagging `.formatted()` YAML in `service.generator.**` — not yet enforced as hard failure until phases 1-4 complete.
- **No existing generator, contributor, or support class is modified.** Zero behavior change.

## Capabilities

### New Capabilities

None — no new user-observable behavior. `skip_specs: true`. All new code (records, serializer, test helpers) requires unit test coverage to maintain Codecov thresholds.

### Modified Capabilities

None — `conversion-pipeline`'s "Conversion output is behavior-preserving" requirement already covers all 5 phases.

## Impact

| Area | Impact |
|------|--------|
| **Backend** | New `model/kuadrant/` package (~8 record files, ~200 lines total), new `ManifestSerializer`, new test utility |
| **Tests** | New unit tests required for every new class: all record types (`*ManifestTest`), `ManifestSerializerTest`, `YamlAssertionsTest`. Codecov must not degrade — new lines must be covered. No existing tests modified |
| **API** | None |
| **Frontend** | None |
| **Dependencies** | `jackson-dataformat-yaml`, Fabric8 Istio model artifact added to `pom.xml`. Unblocks phases 1-4 |
| **GitHub** | Phase 0 of [#262](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/262) |
