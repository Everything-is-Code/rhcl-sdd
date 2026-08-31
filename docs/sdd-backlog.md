# SDD Backlog — Migration Toolkit

Prioritized OpenSpec changes mapped to GitHub issues. Use `/opsx-propose` with the **change name** column.

## P0 — Architecture & debt

| Change name | GitHub | Scope | Status |
|-------------|--------|-------|--------|
| `conversion-strategy-registry` | #40 | Strategy + Registry for `ResourceGenerator`; Contributor pattern for HTTPRoute/Policy/Secret | **Archived** — OpenSpec `2026-08-25-conversion-strategy-registry`; PR #190 merged |
| `frontend-component-split` | #41 | Split large pages into `components/`, `AppStateContext`, shared `apiError` | **Archived** — OpenSpec `2026-08-25-frontend-component-split`; PR [#191](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/191) merged |
| `unified-error-envelope` | #171 | Unified error envelope, `ApiException` hierarchy, frontend i18n code mapping | **Archived** — OpenSpec `2026-08-25-unified-error-envelope`; PR [#195](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/195) merged |
| `unified-error-envelope-complete` | #196 | Complete migration: all controllers → envelope, `NotFoundException`, 14 codes, full frontend i18n | **Archived** — OpenSpec `2026-08-25-unified-error-envelope-complete`; PR [#197](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/197) merged |
| `conversion-readme-args` | #170 | Replace `generateReadme(...)` positional note list with structured builder | **Largely addressed on #40 branch** (`ReadmeSupport` + `ReadmeNotes`); close or narrow on #40 merge |
| `export-performance-baseline` | #169 | Pagination, bulk convert parallelism, History UI pagination | Open |

## P0 — Typed YAML generation (#262)

Epic [#262](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/262): replace `String.formatted()` YAML with typed Java objects (Fabric8 models + hand-written records for Kuadrant CRDs). Supersedes #53 (closed). Behavior-preserving: `skip_specs: true` — `conversion-pipeline`'s existing requirements cover all phases.

| Change name | GitHub | Scope | Status |
|-------------|--------|-------|--------|
| `typed-yaml-infra` | #262 | Phase 0 — `model/kuadrant/` records, `ManifestSerializer`, `YamlAssertions`, deps (Jackson YAML, Fabric8 Istio), ArchUnit scaffold | **Archived** — OpenSpec `2026-08-31-typed-yaml-infra`; PR [#263](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/263) |
| `typed-yaml-k8s-gateway` | #262 | Phase 1 — Gateway, ConfigMap, Secret + 5 secret contributors → Fabric8 typed models | Proposed — depends on phase 0 |
| `typed-yaml-istio` | #262 | Phase 2 — ServiceEntry, DestinationRule, 5 EnvoyFilters, Istio AuthorizationPolicy, Telemetry → Fabric8 Istio model | Proposed — depends on phase 0 |
| `typed-yaml-kuadrant` | #262 | Phase 3 (largest) — AuthPolicy (+10 contributors), RateLimitPolicy, TLS/DNSPolicy, APIProduct, ApiKey → Java records | Proposed — depends on phase 0 |
| `typed-yaml-httproute` | #262 | Phase 4 (recommended last) — HTTPRoute + 8 contributors + 3 support → Fabric8 Gateway API model | Proposed — depends on phase 0 |

## P1 — Test coverage (#210) — **closed**

Epic [#210](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/210) **closed** Aug 2026. Codecov `backend` ~37% → ~50–60%; JaCoCo lines ~60% on `main`.

| Change name | GitHub | Scope | Status |
|-------------|--------|-------|--------|
| `test-coverage-controller` | #210 | Tier A: `controller` ≥90% lines | **Archived** — PR [#213](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/213) merged |
| `test-coverage-exception-client` | #210 | Tier: `exception` + `client` | **Archived** — PR [#214](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/214) merged |
| `test-coverage-conversion-support` | #210 | Tier: `service.conversion` | **Archived** — PR [#216](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/216) merged |
| `test-coverage-generators` | #210 | Tier: `service.generator` | **Archived** — PR [#217](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/217) merged |
| `test-coverage-contributors` | #210 | Tier: `service.generator.contributor` | **Archived** — PR [#218](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/218) merged |
| `test-coverage-dto-model-entity` | #210 | Optional PR-6 dto/model/entity | **Archived (superseded)** — baseline on `main`: dto 100%, entity ~65%, model ~50%; merged into #222 |

## P1 — Test coverage follow-up (#222)

| Change name | GitHub | Scope | Status |
|-------------|--------|-------|--------|
| `test-coverage-service-data-layers` | [#222](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/222) | Slices 1–2: model/entity + `ThreeScaleExportService` WireMock | **Archived** — PRs [#224](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/224), [#225](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/225) |
| `test-coverage-cluster-compat` | [#222](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/222) | Slices 3–4: cluster/compat + `ConversionService` orchestration | **Archived** — PR [#226](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/226) |

Open PRs for #222: #224, #225, #226.

## P1 — Policy conversion epic (#149)

Each child issue should become its own change (name pattern: `policy-<3scale-name>-conversion`).

Recognized policies needing conversion (verify issue list on GitHub for current status):

| 3scale policy (typical) | Target resource(s) |
|-------------------------|-------------------|
| cors | HTTPRoute filters |
| header_modification | HTTPRoute |
| url_rewriting | HTTPRoute |
| jwt / api_key / oauth | AuthPolicy, Secret |
| rate_limit | RateLimitPolicy |
| caching | (mapping per #149 spec) |
| soap | compatibility + conversion rules |
| lua (custom) | compatibility scoring; limited conversion |

**Workflow per policy:**

1. `enrich-us` on the child issue
2. `/opsx-propose policy-<name>-conversion`
3. `/opsx-apply` with tests in `ConversionServiceTest`
4. PR referencing issue + change name

## P2 — Program docs migration

| Change name | Scope |
|-------------|-------|
| `rhcl-ai-docs-sunset` | Move active architecture/workflow docs into this store; mark rhcl-ai AI docs as legacy pointers |

## Phase 2 (out of scope for initial store)

| Area | Notes |
|------|-------|
| `from-3scale-to-connectivity-link` | Adapter repo SDD — separate store or extend this one |
| E2E lab automation | Playwright MCP optional |
| rhcl-ai replacement | Full sunset of cross-repo skills into `ai-specs/skills/` |

## Engram migration

Legacy SDD used Engram topic keys `sdd/{change-name}/*`. New work uses OpenSpec only. When reviving an Engram change, re-propose via `/opsx-propose` using archived gateforge `docs/archive/*.md` as input context.
