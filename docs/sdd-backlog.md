# SDD Backlog — Migration Toolkit

Prioritized OpenSpec changes mapped to GitHub issues. Use `/opsx-propose` with the **change name** column.

## P0 — Architecture & debt

| Change name | GitHub | Scope | Status |
|-------------|--------|-------|--------|
| `conversion-strategy-registry` | #40 | Strategy + Registry for `ResourceGenerator`; Contributor pattern for HTTPRoute/Policy/Secret | **Archived** — OpenSpec `2026-08-25-conversion-strategy-registry`; PR #190 merge pending |
| `frontend-component-split` | #41 | Split large pages into `components/`, `AppStateContext`, shared `apiError` | **Archived** — OpenSpec `2026-08-25-frontend-component-split`; PR [#191](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/191) merge pending |
| `unified-error-envelope` | #171 | Unified error envelope, `ApiException` hierarchy, frontend i18n code mapping | **Archived** — OpenSpec `2026-08-25-unified-error-envelope`; PR [#195](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/195) merge pending |
| `unified-error-envelope-complete` | #196 | Complete migration: all controllers → envelope, `NotFoundException`, 14 codes, full frontend i18n | **Archived** — OpenSpec `2026-08-25-unified-error-envelope-complete`; PR [#197](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/197) merge pending |
| `conversion-readme-args` | #170 | Replace `generateReadme(...)` positional note list with structured builder | **Largely addressed on #40 branch** (`ReadmeSupport` + `ReadmeNotes`); close or narrow on #40 merge |
| `export-performance-baseline` | #169 | Pagination, bulk convert parallelism, History UI pagination | Open |

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
