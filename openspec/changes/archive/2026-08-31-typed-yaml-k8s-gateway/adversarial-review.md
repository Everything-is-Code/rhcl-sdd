## Adversarial review

**Scope:** `typed-yaml-k8s-gateway` / [#262](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/262) phase 1  
**Branch:** `feature/262-typed-yaml-k8s-gateway` (stacked on `feature/262-typed-yaml-infra` / PR #263)  
**Sources:** `openspec/changes/typed-yaml-k8s-gateway/{proposal,design,tasks}.md`, `verify-report.md`, `review-report.md`, product diff (`GatewayGenerator`, `ConfigMapGenerator`, `SecretGenerator`, `SecretBuilder`, 5 secret contributors, `ConversionYamlSupport`, tests)  
**Tests reviewed against:** `mvn test` — 1016 tests, 0 failures (post adversarial fixes, 2026-08-31)

### Spec alignment

| Criterion | Adversarial check | Result |
|-----------|-------------------|--------|
| Replace `.formatted()` in 8 target classes | ArchUnit allowlist shrunk; grep shows no `.formatted()` in migrated classes | **Met** |
| Serialize via `ManifestSerializer` | Generators inject serializer; `SecretBuilder.build(ManifestSerializer)` used from `SecretGenerator`. Static helpers call `build()` → `new ManifestSerializer()` (same as phase 0 fallback pattern) | **Met** (minor CDI bypass in static paths) |
| Structural `YamlAssertions` tests | Generator + `ConversionServiceTest` gateway paths migrated | **Met** |
| Preserve secret `stringData` semantics | Contributors still use `stringData` via Fabric8; placeholders unchanged | **Met** |
| `includeMigratedFromLabel` opt-out | Typed generators skip at source; regex strip extended for quoted + flow-style (paired entries) | **Mostly met** (gap: lone flow-style label) |
| Behavior-preserving output | Fabric8 field order differs; semantically equivalent for K8s apply. Discovery marker now real annotation (improvement, not byte-identical) | **Acceptable** with PR note |

### Findings

| Severity | Area | Finding | Evidence | Fix (code/spec/tests) |
|----------|------|---------|----------|------------------------|
| Minor | `ConversionYamlSupport` | Flow-style strip does not handle a labels map containing **only** `migrated-from: 3scale` (no trailing comma). Regexes target `key, ` or `, key` forms. | `stripMigratedFromLabel` implementation; test covers `{app: demo, migrated-from: 3scale}` only | **Fixed** — `labels: {migrated-from: …}` → `labels: {}`; test `stripMigratedFromLabel_removesSoleFlowStyleLabel` |
| Minor | `SecretBuilder` | `injectYamlCommentBeforeStringData` returns YAML unchanged if `stringData:` absent — WARNING comments silently dropped in that edge case. | `SecretBuilder.java` lines 136–140; mitigated today by empty-map Fabric8 path + `injectEmptyStringData` fallback | **Fixed** — fallback injects before `type:` or appends; test `injectYamlComment_fallsBackBeforeTypeWhenStringDataAbsent` |
| Minor | `SecretBuilder` | `injectEmptyStringData` matches literal `type: Opaque` — breaks if Fabric8 quotes type (`type: "Opaque"`). | `injectEmptyStringData` | **Fixed** — `indexOfOpaqueType` handles quoted forms; test `injectEmptyStringData_handlesQuotedOpaqueType` |
| Minor | Design §2 | `SecretBuilder.build()` no-arg and generator `serializer()` fallbacks instantiate `new ManifestSerializer()` outside CDI. | `SecretBuilder.build()`, `GatewayGenerator.serializer()` | **Documented** — `resolveSerializer()` helper with javadoc; pattern unchanged (testability) |
| Minor | Mixed migration | `includeMigratedFromLabel=false` still depends on regex strip for ~30 string-format generators on ArchUnit allowlist; typed tier is source-clean. | `ConversionService.convert`, `ArchitectureTest` allowlist | Expected until later phases; no action for phase 1 |
| Minor | Operations | Stacked PR bundles phase 0 infra + phase 1 — review/merge/Codecov attribution risk. | Branch base `feature/262-typed-yaml-infra` | Merge #263 first; clear PR sections |
| Question | Product | Discovery marker moved from YAML injection to `metadata.annotations` — any external tooling parsed the old inline annotation line? | `SecretContributorDiscoveryTest`, `SecretBuilderTest` | Maintainer confirm; tests lock new behavior |
| Question | CI | Windows CRLF can false-fail whitespace-sensitive YAML asserts; Linux CI is source of truth. | `AGENTS.md` convention | Confirm CI green on push |

**No Blocker or Major findings** after code-review follow-ups (explicit `apiVersion`/`kind`, label tests, discovery-marker logging, flow-style strip for paired labels).

### Attack scenarios exercised (mental / test-backed)

1. **Empty App ID credentials** — empty `stringData: {}` + `# WARNING` comment preserved (`AppIdKeySecretContributorTest`, `ConversionServiceTest`).
2. **Malformed discovery marker** — ignored with WARN log; no annotation (`SecretBuilderTest`).
3. **Discovery marker value with colons** — full value retained (`SecretBuilderTest`).
4. **`includeMigratedFromLabel=false`** — label absent on Gateway, ConfigMap, Secret (`*GeneratorTest`, `ConversionServiceTest`).
5. **Duplicate `stringData` key** — last-write-wins (`SecretBuilderTest`).
6. **Random API key generation** — unchanged behavior (`ApiKeySecretContributor`); not a regression.
7. **Credential leakage in logs** — contributor WARNs reference service IDs, not secret values; no new logging of `stringData`.

### Verdict

**PASS (adversarial)** — code gaps from initial review fixed; remaining items are operational (stacked PR / CI) or maintainer confirmation.

### Before archive

- [ ] Push branch and confirm **Linux CI** green (Codecov patch on stacked diff).
- [ ] Merge **PR #263** (phase 0) before retargeting phase 1 PR to `main`.
- [ ] PR description: call out Fabric8 YAML ordering, discovery-marker annotation change, and `includeMigratedFromLabel` dual path (source + strip).
- [ ] Maintainer confirm: no external tooling depended on inline discovery-marker YAML injection.
