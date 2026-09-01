## Adversarial review

**Scope:** `typed-yaml-istio` / [#262](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/262) phase 2  
**Sources:** `openspec/changes/typed-yaml-istio/{proposal,design,tasks}.md`, `verify-report.md`, branch `feature/262-typed-yaml-istio` diff vs stacked base

### Spec alignment

| Criterion | Status |
|-----------|--------|
| 9 generators off `String.formatted()` | Met — ArchUnit allowlist entries removed |
| Fabric8 builders for ServiceEntry, DestinationRule, AuthorizationPolicy, Telemetry | Met |
| EnvoyFilter via typed envelope + maps for dynamic patches | Met per `design.md` #1 |
| Lua bodies unchanged (`escapeLua`, script logic) | Met — MaintenanceMode still uses `String.format` for inline code |
| Serialization via `ManifestSerializer` | Met |
| Tests → `YamlAssertions` / structural parse | Met (9 generator test classes + `ConversionServiceTest` adjustments) |
| Behavior-preserving (proposal) | **Semantic** pass — not byte-identical YAML |
| Non-goals: no Kuadrant/HTTPRoute/core K8s | Honored |

### Findings

| Severity | Area | Finding | Evidence | Fix (code/spec/tests) |
|----------|------|---------|----------|------------------------|
| ~~Major~~ | Multi-doc YAML | `joinYamlChunks` glued `---` to prior line → invalid YAML, `MigrationWorkflowIT` failures | E2E verify run; `resolution: DNS---` pattern | **Fixed:** newline before `---` in `IstioManifestSupport.joinYamlChunks`; `IstioManifestSupportTest` added |
| Minor | Validation | `security.istio.io/v1` absent from `KNOWN_CRDS` → false unknown-CRD warnings for AuthorizationPolicy | `ValidationService.java` | **Fixed:** added to `KNOWN_CRDS` |
| Minor | Hygiene | Duplicate imports in `ServiceEntryGenerator`, `DestinationRuleGenerator` | grep on imports | **Fixed** |
| Minor | Hygiene | `backend/cp.txt` debug artifact (40KB classpath dump) | untracked file | **Fixed:** deleted |
| Minor | Tests | No dedicated `joinYamlChunks` test before review — E2E-only detection | verify gap | **Fixed:** `IstioManifestSupportTest` |
| Minor | Tests | `includeMigratedFromLabel=false` not covered per Istio generator | phase 1 had Gateway test | **Partial fix:** `ServiceEntryGeneratorTest`; others rely on `ConversionServiceTest` |
| Question | Output fidelity | Fabric8 field order / quoting may differ from pre-migration strings | Jackson serialization | Accept for refactor; document in PR — not a functional bug if tests green |
| Question | EnvoyFilter | Map patches bypass builder validation — malformed patch could still serialize | `EnvoyFilterManifests` | Mitigated by existing generator tests + design acceptance; no change required |

### Adversarial scenarios exercised (post-fix)

- **Multi-doc ServiceEntry + ExternalName Service:** valid parse via `parseDocuments` — PASS  
- **MaintenanceMode Lua with `---`, `#`, quotes:** edge case 4.3 — PASS  
- **Empty / disabled policies:** `isConfigEnabled` null-safe paths — PASS  
- **ArchUnit:** no `.formatted()` in migrated generator classes — PASS  
- **CRLF false positive:** structural assertions, not raw string equality on full YAML blobs — low risk  
- **Secrets in YAML/logs:** N/A — Istio phase does not touch secret generators  
- **#40 / #170:** no `ConversionService` or Readme API changes — PASS  

### Verdict

**PASS WITH GAPS** — no blockers or open majors after `joinYamlChunks` fix and review-time hygiene. Remaining gaps are minor (per-generator `includeMigratedFromLabel` tests, byte-format nuance documentation).

### Before archive

- [x] `mvn verify` green after multi-doc fix  
- [x] Review-time fixes committed with implementation (imports, `cp.txt`, `KNOWN_CRDS`, tests)  
- [ ] Maintainer review on stacked PR chain (#263 → #264 → phase 2)  
- [ ] CI Codecov on PR — confirm no regression  
- [ ] PR body notes semantic-vs-byte YAML preservation and `joinYamlChunks` E2E catch  
