## Context

See `proposal.md` for motivation. Post-#40, generators/contributors are well-separated (one class per output file or contributor), but each still builds YAML via `.formatted()` text blocks. The target: every K8s resource is a typed Java object — Fabric8 model for standard/Istio resources, hand-written record for Kuadrant CRDs.

## Goals / Non-Goals

**Goals:**
- Provide the complete `model/kuadrant/` record package so phase 3 can immediately use typed constructors.
- Provide `ManifestSerializer` so all phases use one serialization path (no ad-hoc `Yaml.dump()`/`Serialization.asYaml()` scattered per class).
- Provide `YamlAssertions` so every phase can replace `assertTrue(yaml.contains(...))` with structural checks.
- Resolve all dependency questions (Fabric8 Istio, Gateway API, Jackson YAML) in this phase, not mid-migration.

**Non-Goals:**
- Migrating any actual generator or contributor (phases 1-4).
- Enforcing the ArchUnit rule as a hard failure (only after all phases land).

## Decisions

**1. Jackson YAML (`jackson-dataformat-yaml`) for record serialization, not SnakeYAML.**
Rationale: Fabric8's own `Serialization.asYaml()` uses Jackson internally — using the same serializer for custom records ensures consistent output formatting (indentation style, null handling, quoting). Quarkus already ships Jackson core; only the YAML dataformat module is new. SnakeYAML remains as a transitive dependency (Jackson YAML uses it) but we don't call it directly.

**2. Records model only the fields we actually emit, not the full CRD schema.**
Rationale: each Kuadrant CRD has 50+ fields in the upstream schema; we emit 5-15. Modeling only what we use keeps records small (~10-30 lines each), avoids upstream coupling, and makes compile errors meaningful (a typo in a field name means the field doesn't exist, not that it's one of 50 options).

**3. `ManifestMeta` record instead of reusing Fabric8 `ObjectMeta`.**
Rationale: Fabric8 `ObjectMeta` has 40+ fields (finalizers, managedFields, ownerReferences...) that we never populate. A minimal record (`name`, `namespace`, `labels: Map<String,String>`, optional `annotations`) serializes cleanly without null-field noise. Jackson's `@JsonInclude(NON_NULL)` keeps output concise.

**4. `ManifestSerializer` is a CDI `@ApplicationScoped` bean, not a static utility.**
Rationale: allows injecting the pre-configured `ObjectMapper` (with YAML factory, `NON_NULL` inclusion, field ordering) once at startup rather than recreating it per call. Also testable via `@QuarkusTest` injection.

**5. Records use `@JsonPropertyOrder` to guarantee `apiVersion`, `kind`, `metadata`, `spec` ordering.**
Rationale: YAML is order-insensitive for correctness, but deterministic output order makes diffs readable and test assertions stable. Jackson respects `@JsonPropertyOrder`; the annotation is a one-liner per record.

**6. ArchUnit rule uses an explicit shrinking allowlist, not a growing blocklist.**
Rationale: allowlist makes progress visible — each phase removes its migrated files from the list. When the list is empty (after phase 4), the rule becomes a hard guard. A blocklist approach would grow stale as new files are added.

## Risks / Trade-offs

- **[Risk]** Kuadrant record field structure might not perfectly match what a generator currently emits (e.g. nested `spec.rules.authentication` in AuthPolicy is deeply nested). → **Mitigation**: design nested records during this phase by reading the actual `.formatted()` templates in the 6 Kuadrant generators to extract the exact field structure they emit; validate by writing unit tests that serialize a sample record and compare to expected YAML.
- **[Risk]** Jackson YAML default formatting may differ from current hand-written YAML formatting (e.g. flow vs block style for empty maps). → **Mitigation**: configure `ObjectMapper` with `MINIMIZE_QUOTES`, `WRITE_DOC_START_MARKER: false`, and block-style collections. Validate in `ManifestSerializerTest`.
- **[Risk]** `ManifestMeta` may need `annotations` for some generators (e.g. `HttpRouteAnnotationsContributor`) but not others. → **Mitigation**: make `annotations` an `Optional<Map<String,String>>` or `@JsonInclude(NON_NULL)` nullable field — only serialized when populated.

**7. RateLimit `# WARNING` YAML comments are not modeled in records.**
Rationale: `RateLimitSupport` embeds migration-warning comment lines inside `spec.limits` blocks. Jackson record serialization cannot round-trip YAML comments. Phase 3 (`typed-yaml-kuadrant`) will preserve the same **structural** limits (rates, windows, limit names) via typed records; comment text is already duplicated in README notes where applicable. Accept comment loss in manifest YAML or add a custom serializer in phase 3 if product requires inline warnings — equivalence tests compare parsed structure (comments ignored by the YAML parser).
