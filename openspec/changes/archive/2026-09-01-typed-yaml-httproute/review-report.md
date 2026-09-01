## AI Code Review

_Disclaimer: AI-generated review. Human maintainer review required._

**Change:** typed-yaml-httproute | **Issue:** #262 (phase 4 — final) | **PR:** not created yet

**Review basis:** uncommitted WIP on branch `feature/262-typed-yaml-httproute` (rebased on kuadrant `c2fba05`) in worktree `migration-toolkit-rhcl-worktrees/262-httproute`. Mixed staged/unstaged state (`MM` on `CorsFiltersContributor.java`, `HttpRouteGeneratorTest.java`). No commit yet.

**Tests run:** `mvn test` in worktree backend — exit 0 (Windows). Linux CI not verified in this review.

### Blocker

- **ArchUnit allowlist not emptied — epic-closing criterion unmet.** `ArchitectureTest.java` was not modified. `FORMATTED_YAML_GENERATOR_ALLOWLIST` still lists 7 classes (`CorsOptionsContributor`, `HttpRouteAnnotationsContributor`, `HttpRouteBuilder`, `MappingRulesContributor`, `RetryContributor`, `RoutingContributor`, `UpstreamContributor`). `FORMATTED_YAML_CONVERSION_ALLOWLIST` still lists 4 (`HttpRouteSupport`, `JwtClaimCheckSupport`, `RoutingSupport`, `UpstreamSupport`). Proposal and tasks 4.7–4.8 require an **empty** allowlist when phase 4 completes (only `JwtClaimCheckSupport` may remain for Markdown). Migrated classes no longer call `String.formatted()` for YAML — the allowlist is stale, not shrunk.

### Major

- **`HttpRouteContributorDiscoveryTest` was changed** despite proposal stating it should pass unmodified. Assertion now splits `x-discovery-marker` and `rhcl-httproute-test` instead of checking `TestMarkerHttpRouteContributor.MARKER` as a single substring. Functionally reasonable for Fabric8 double-quoting, but violates the explicit acceptance criterion and reduces coupling to the marker constant.
- **`HttpRouteBuilder.injectCorsFilters` is fragile post-serialization string surgery** with no dedicated unit tests. Native CORS (`corsNative=true`) relies on replacing `    matches:\n` / prepending to `    filters:\n`, assuming Fabric8 always serializes rule fields alphabetically (`backendRefs → filters → matches`). `ConversionServiceTest` covers end-to-end native CORS, but a Fabric8 serializer change could break injection silently; `injectCorsFilters` itself is untested.
- **Edge-case test matrix from tasks §5 largely incomplete.** Only partial coverage: 5.1 (`build_withNoRules_producesValidHttpRoute`), 5.5 (`contribute_nativeCors_emptyOrigins_useWildcard`). Missing or absent for HttpRoute scope: duplicate `addAnnotation` last-write-wins (5.2), null/empty upstream URL (5.4), invalid header chars (5.6), zero timeout `0s` (5.7), empty/null URL rewrite path (5.8), same-`@Priority` stable ordering (5.9), `@Priority` filter/rule order regression test (4.6).
- **`HeaderModContributor` liquid-template handling changed observable output.** Liquid headers previously emitted a YAML comment (`# Header '…' uses liquid template — manual conversion required`). Now only `LOG.infof` — generated `httproute.yaml` no longer carries the operator-visible hint. Tests assert filter absence, not log or YAML comment parity.
- **`buildBackendRefs` weight default semantics narrowed.** Old string formatter emitted `weight: 1` for every backend when multiple backends were selected and `weight` was null. New typed builder only sets weight when `weighted && backend.weight != null`. Gateway API may default unspecified weight to 1 at runtime, but YAML output differs; no test covers multi-backend with null weights.

### Moderate

- **Regression assertions weakened to substring checks** in `ConversionServiceTest`, `HttpRouteGeneratorTest`, and `ConversionSupportQuarkusTest` (e.g. `PathPrefix` instead of `type: PathPrefix`, `GET` instead of `method: GET`, header names without `name:` prefix). Reduces false negatives from Fabric8 quoting but increases false positives (e.g. `GET` could match unrelated fields). Acceptable trade-off if documented; still a coverage-quality regression.
- **`RoutingSupport` / `UpstreamSupport` not migrated to typed Gateway API objects** as design §4 states — only `String.format` substitution for Markdown readme warnings. Allowlist entries for these classes are unnecessary once shrunk; scope is readme-only, not httproute YAML.
- **`CorsFiltersContributor` native CORS path still builds YAML via `StringBuilder`** (`buildNativeCorsFilterYaml`). Consistent with design fallback for non-standard `type: CORS`, but couples native mode to manual indentation and duplicates CORS config parsing alongside `buildCorsResponseHeaderFilter`.
- **`tasks.md` checklist entirely unchecked** — no `verify-report.md`; `/verify` not recorded for this phase.
- **WIP hygiene:** unstaged diff adds `import java.util.Map` to `CorsFiltersContributor` and further softens `HttpRouteGeneratorTest` assertions beyond staged diff. Should be consolidated before commit.

### Minor

- **`HttpRouteBuilder.build()` with zero rules** omits `rules:` key entirely (valid HTTPRoute) rather than `rules: []` mentioned in task 5.1 — behavior is acceptable; task wording may be imprecise.
- **`MappingRulesContributor` with no mapping rules** emits catch-all `/` rule (by design), not a no-op — task 5.3 wording differs from product behavior; existing test `shouldContribute_false_whenNoRules_usesCatchAll` documents catch-all correctly.
- **`TimeoutsContributor.toTimeouts`** uses `connectRaw + "s"` / `readRaw + "s"` on `Object` — works for numeric types but would produce odd strings for non-numeric config (pre-existing pattern).

### Nit

- Duplicate CORS configuration parsing blocks in `CorsFiltersContributor` (`buildNativeCorsFilterYaml` vs `buildCorsResponseHeaderFilter`) — opportunity to extract shared config normalization later.
- `HttpRouteBuilder.resolveSerializer()` creates a new `ManifestSerializer` per call in CDI-free paths — pre-existing pattern, unchanged risk.

### Summary

Typed Fabric8 migration for httproute contributors and support classes is largely implemented and **local `mvn test` is green**, with `@Priority` values preserved on all HttpRoute contributors. The change **cannot merge as the #262 closing phase** until `ArchitectureTest` allowlists are emptied (Blocker). Secondary risks: untested `injectCorsFilters`, incomplete edge-case matrix, liquid-header YAML comment regression, and weakened substring assertions. Consolidate WIP unstaged changes before PR.
