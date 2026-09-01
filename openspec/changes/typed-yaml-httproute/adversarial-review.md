## Adversarial Review

**Scope:** OpenSpec change `typed-yaml-httproute` — GitHub #262 phase 4 (final)

**Sources:** `proposal.md`, `design.md`, `tasks.md`; product diff `git diff HEAD` (29 files, +793/−459) on WIP branch `feature/262-typed-yaml-httproute` in worktree `262-httproute` (no commit; mixed staged/unstaged).

**Tests:** `mvn test` — exit 0 (Windows local). Linux CI not run in this review.

### Spec alignment

- [ ] **ArchUnit allowlist EMPTY when phase complete (JwtClaimCheckSupport only if needed)** — **FAIL** — `ArchitectureTest.java` unchanged; generator allowlist has 7 entries, conversion allowlist has 4 (`JwtClaimCheckSupport` + 3 migrated httproute classes). Epic-closing signal not achieved.
- [ ] **`@Priority` ordering preserved** — **PASS** — HttpRoute contributor priorities unchanged: 100 (`HttpRouteAnnotationsContributor`), 200 (`HeaderModContributor`), 250 (`CorsFiltersContributor`), 300 (`TimeoutsContributor`), 310 (`RetryContributor`), 350 (`RoutingContributor`), 360 (`UpstreamContributor`), 400 (`MappingRulesContributor`), 500 (`CorsOptionsContributor`).
- [ ] **`HttpRouteContributorDiscoveryTest` unchanged** — **FAIL** — Test modified: discovery assertion split into key + value substrings for Fabric8 quoting instead of `TestMarkerHttpRouteContributor.MARKER` constant check.
- [ ] **Behavior-preserving migration (typed Fabric8 + `ManifestSerializer`)** — **PARTIAL** — Core contributors/builder/generator migrated; native CORS uses post-serialize YAML injection; liquid header hints removed from YAML output; multi-backend null-weight YAML may differ; local regression suite green.
- [ ] **`HttpRouteBuilder` wraps Fabric8 `HTTPRouteBuilder`** — **PASS** — `HttpRouteBuilder.build()` constructs `io.fabric8.kubernetes.api.model.gatewayapi.v1.HTTPRouteBuilder`, serializes via `ManifestSerializer`.
- [ ] **8 contributors migrated to typed objects** — **PASS** — `MappingRulesContributor`, `RoutingContributor`, `UpstreamContributor`, `CorsOptionsContributor`, `RetryContributor`, `TimeoutsContributor`, `HeaderModContributor`, `HttpRouteAnnotationsContributor` use Fabric8 Gateway API types. (`CorsFiltersContributor` also refactored; not in proposal's 8-name list but part of httproute assembly.)
- [ ] **3 support classes migrated** — **PARTIAL** — `HttpRouteSupport` returns `HTTPBackendRef` / `HTTPRouteFilter` lists. `RoutingSupport` / `UpstreamSupport` only swapped `formatted()` → `String.format()` for Markdown readme — no typed httproute objects.
- [ ] **Serialization through `ManifestSerializer`** — **PASS** — `HttpRouteGenerator.generate()` → `builder.build(serializer)`; `HttpRouteBuilder.build(ManifestSerializer)` default path.
- [ ] **Tasks §4.6 `@Priority` order test** — **FAIL** — No test asserting filter/rule array order against #40 baseline.
- [ ] **Tasks §5 edge-case / unhappy-path tests** — **PARTIAL** — ~2 of 10 items substantively covered; remainder open.
- [ ] **Tasks §6 `ConversionServiceTest` httproute assertions** — **PARTIAL** — Assertions updated for Fabric8 output; many weakened to substring checks; suite passes locally.
- [ ] **Tasks §6.2 `ConversionServiceConcurrencyTest`** — **PASS (assumed)** — Included in full `mvn test` run (exit 0); no isolated confirmation in review log.

### Findings

| Severity | Area | Finding | Evidence | Fix (code/spec/tests) |
|----------|------|---------|----------|------------------------|
| Blocker | ArchUnit | Allowlist not shrunk — #262 structural goal unmet | `ArchitectureTest.java` lines 207–223 unchanged; 11 allowlisted classes | Remove all migrated classes from both allowlists; leave only `JwtClaimCheckSupport` (Markdown). Run `ArchitectureTest`. |
| Major | CORS / builder | `injectCorsFilters` string-replace on serialized YAML — brittle | `HttpRouteBuilder.injectCorsFilters` (lines 190–211); no unit tests | Add `HttpRouteBuilderTest` cases for inject with/without existing filters, multiple rules; or model native CORS without post-processing. |
| Major | Output parity | Liquid header mod hints removed from YAML | `HeaderModContributor` — `LOG.infof` replaces YAML `# … manual conversion` comment | Restore YAML annotation/comment in output or document intentional change; add regression test if YAML hints required. |
| Major | Tests | Edge-case matrix §5 mostly missing | `tasks.md` §5.2–5.4, 5.6–5.9, §4.6 | Implement targeted tests per task list or track in follow-up with explicit waiver. |
| Major | Spec | `HttpRouteContributorDiscoveryTest` modified vs proposal | Diff: assertion uses split key/value vs `MARKER` constant | Revert to constant-based check with Fabric8-safe matcher, or update proposal/spec to allow quoting adaptation. |
| Major | Backend refs | Null weight omitted for multi-backend rules | `HttpRouteSupport.buildBackendRefs` — weight only if `!= null` | Add test for multi-backend null weights; align with prior `weight: 1` default if YAML parity required. |
| Moderate | Tests | Substring assertions weaker | `ConversionServiceTest`, `HttpRouteGeneratorTest`, `ConversionSupportQuarkusTest` diffs | Prefer `YamlAssertions` / structured parsing where possible. |
| Moderate | CORS | Native CORS still string-built | `CorsFiltersContributor.buildNativeCorsFilterYaml` | Accept as documented fallback; consolidate duplicate config parsing. |
| Moderate | Process | `tasks.md` unchecked; no verify report | All task checkboxes `[ ]` | Run `/verify`, record in `verify-report.md`, check off completed items. |
| Moderate | WIP | Unstaged changes on top of staged | `git status` `MM` files | Stage and commit atomically before PR. |
| Minor | Support | `RoutingSupport`/`UpstreamSupport` readme-only change | Diff: `String.format` only | Remove from conversion allowlist when shrinking; no httproute typed migration needed. |

### Adversarial scenarios (how acceptance criteria could still fail)

| Scenario | Likelihood | Mitigation status |
|----------|------------|-------------------|
| Fabric8 changes HTTPRoute field order/indentation → native CORS injection corrupts YAML | Low–medium | End-to-end CORS tests pass today; no direct `injectCorsFilters` tests |
| Multi-backend collision with null `backend.weight` → unequal traffic split vs pre-migration YAML | Low | No explicit test; runtime may still default weight 1 |
| Liquid header policy → operator sees clean YAML with no migration hint | Medium | Behavior change; log-only now |
| Weakened `contains("GET")` assertions pass when method misplaced | Low | Full suite green; risk on future regressions |
| New `.formatted()` in generator package without allowlist update | Low post-merge if allowlist emptied | Blocked today by stale allowlist |
| Concurrent convert requests sharing `HttpRouteBuilder` | Low | Stateless per-request builder; concurrency test assumed green |

### Verdict

**FAIL**

Core typed migration is implemented and local tests pass, but the phase **does not satisfy explicit closing criteria**: ArchUnit allowlist is not empty, discovery test was modified, edge-case and ordering tests are incomplete, and observable YAML parity gaps (liquid headers, optional weight omission) remain unverified.

### Before archive

1. Empty `ArchitectureTest` allowlists (retain `JwtClaimCheckSupport` only in conversion set).
2. Resolve `HttpRouteContributorDiscoveryTest` vs proposal (revert or update spec).
3. Add `injectCorsFilters` unit tests or eliminate post-serialize injection.
4. Complete or explicitly defer tasks §4.6 and §5 with tracked follow-up issue.
5. Run `/verify` on Linux CI; record `verify-report.md`.
6. Consolidate unstaged WIP; commit; open PR noting #262 phase 4 / allowlist empty as epic closure.
7. Comment on #262 only after merge and allowlist verified empty.
