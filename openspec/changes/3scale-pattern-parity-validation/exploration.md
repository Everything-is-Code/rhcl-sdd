## Exploration: 3scale-pattern-parity-validation

**Change name:** `3scale-pattern-parity-validation`  
**Store:** `rhcl-sdd` + Engram `migration-toolkit-rhcl`  
**Epic context:** #149 (policies), #202 (E2E CI), existing `testdata/seed/catalog.yaml`

### Current State

The repo already has the **building blocks** for pattern-level validation, but they are **not wired into one automated matrix**:

| Layer | What exists | Gap |
|-------|-------------|-----|
| **Seed catalog** | `testdata/seed/catalog.yaml` — 17 `rhcl_seed_*` products (1 policy/pattern each) + `catalog.yaml` `coverage:` tags | No automated test reads `coverage:`; no export JSON checked into repo |
| **Seeding** | `scripts/seed-rhcl-cases.sh` → `threescale-seed --fixtures catalog.yaml` | Lab-only; not in CI |
| **Backend unit** | ~100 `convert_*` tests in `ConversionServiceTest` + contributor/generator tests | Synthetic `ApiService` builders; **not** tied to seed `system_name` |
| **Backend IT** | `MigrationWorkflowIT` — synthetic jwt/apiKey flows, no 3scale export | No per-seed-product coverage |
| **Frontend E2E** | `frontend/e2e/migration-workflow.spec.ts` + `yaml-expectations.ts` | Only **5** products; **drift** vs catalog (E2E has `rhcl_seed_claim_role_chain`, `auth_chain`, etc. **not** in `catalog.yaml`) |
| **Validation** | `ValidationService` — syntax/structure checks on generated YAML | Not run per seed case in CI matrix |
| **Runtime behavior** | None | No dual-path HTTP parity (3scale APIcast vs RHCL gateway) |

**Interpretation of "same behavior":** today the toolkit only proves **configuration emission** (YAML fragments). True **runtime parity** (status codes, headers, rate limits, auth rejection) needs a cluster + upstream mock and is a separate, heavier program.

### Affected Areas

- `testdata/seed/catalog.yaml` — source of truth for products/policies
- `testdata/seed/README.md` — manual flow docs
- `testdata/exports/` — **new** frozen Admin API export JSON per product (proposed)
- `backend/src/test/e2e/.../SeedCatalogConversionIT.java` — **new** parameterized IT (proposed)
- `backend/src/test/java/.../service/ConversionServiceTest.java` — reference patterns, not seed-linked today
- `frontend/e2e/yaml-expectations.ts` — per-product YAML fragment contract
- `frontend/e2e/migration-workflow.spec.ts` — loops `SEED_YAML_EXPECTATIONS`
- `frontend/e2e/yaml-assertions.ts` — shared assert helper
- `.github/workflows/pr-checks.yml` — PR gate (E2E excluded per #202)
- `documentation/conversion-architecture.md` — output file ↔ policy mapping

### Approaches

1. **Catalog-driven config parity (CI-safe, recommended P1)** — Commit frozen 3scale export fixtures + shared expectation manifest; parameterized backend test: `fixture → ApiService → convert → assert YAML + ValidationService`.
   - Pros: No lab secrets in PR CI; fast; maps 1:1 to `rhcl_seed_*`; catches regressions on every policy PR
   - Cons: Does not prove runtime behavior; fixtures must be refreshed when seeder/policy defaults change
   - Effort: **Medium** (1–2 sprints for harness + first 17 cases)

2. **Extend Playwright lab E2E (live 3scale, recommended P2)** — Align `yaml-expectations.ts` with full catalog; run against seeded tenant on schedule (nightly) or manual `verify-fe`.
   - Pros: Exercises real export path (Admin API → UI → convert); catches integration drift
   - Cons: Needs `THREESCALE_*` secrets; flaky/slow; #202 still open for PR gating
   - Effort: **Low–Medium** per product once harness exists

3. **Runtime parity on OCP/Kind (RHCL apply + HTTP probes, P3)** — Apply generated package to lab cluster; send requests through 3scale APIcast (seed tenant) and RHCL route; compare status/headers/body against httpbin upstream.
   - Pros: Closest to "behaves the same"
   - Cons: Infra-heavy (Kuadrant, Gateway, OIDC stubs); OIDC products need mock IdP; not PR-friendly
   - Effort: **High** — split by policy class (routing vs auth vs rate limit)

4. **Golden-file YAML only (anti-pattern here)** — Full `gateway.yaml` snapshot per product in git.
   - Pros: Simple diff on change
   - Cons: Brittle (namespaces, timestamps, ordering); fights #40 contributor architecture; poor review signal
   - Effort: Low initially, **high maintenance**
   - **Not recommended** as primary strategy; use **fragment contracts** (current `yaml-expectations` style)

### Recommendation

**Phased program — config parity first, runtime parity later.**

**P1 (team next):** Issue epic + Issues #A–#F below — catalog manifest, frozen exports, shared expectations, parameterized `SeedCatalogConversionIT`, drift guard test, PR CI job.

**P2:** Full Playwright matrix + nightly workflow (#202 decision: nightly, not PR).

**P3:** Runtime parity epic scoped by **policy class** (start with routing/CORS/headers — no IdP; defer OIDC/JWT to mock-IdP milestone).

**Immediate hygiene:** Reconcile E2E `yaml-expectations.ts` with `catalog.yaml` (remove or re-add missing chain products to catalog).

### Proposed GitHub Issues (for team backlog)

#### Epic (parent)

**Title:** `test: 3scale seed catalog → RHCL pattern parity matrix`  
**Labels:** `epic`, `area/testing`, `testing`  
**Body summary:** Automated validation that each `rhcl_seed_*` product in `testdata/seed/catalog.yaml` produces the expected RHCL configuration. Phase 1 = CI-safe config parity; Phase 2 = lab E2E; Phase 3 = runtime parity on cluster.

---

#### Issue 1 — Catalog manifest & drift guard (foundation)

**Title:** `test: seed catalog manifest + drift guard (catalog ↔ expectations ↔ tests)`  
**Scope:**
- Add `testdata/seed/manifest.yaml` (or generate from `catalog.yaml`) listing each `system_name`, policy, expected output files, linked test ids
- JUnit/Vitest test fails if: catalog product missing from manifest; E2E expectations reference unknown `system_name`; catalog `coverage:` entry has no test row
- Document in `testdata/seed/README.md`

**AC:** CI fails on catalog/E2E drift; manifest lists all 17 products

---

#### Issue 2 — Frozen 3scale export fixtures

**Title:** `test: commit frozen Admin API export JSON per rhcl_seed_* product`  
**Scope:**
- `testdata/exports/<system_name>.json` — one file per catalog product (from `threescale-seed` + live export or toolbox)
- Script `scripts/refresh-seed-exports.sh` (documented, manual; uses env vars)
- README: when to refresh (policy default change in `3scaleextract`)

**AC:** 17 JSON files; script documented; no secrets in fixtures

---

#### Issue 3 — Backend parameterized seed conversion IT

**Title:** `test: SeedCatalogConversionIT — fixture → convert → YAML contract`  
**Scope:**
- `@QuarkusTest` `SeedCatalogConversionIT` parameterized over manifest rows
- Load export JSON → `ApiService` (reuse export deserializer if exists, else mapper)
- `conversionService.convert` + `validationService.validate`
- Assert per-product **fragment contract** (shared with FE — see Issue 4)
- Runs in `mvn verify` / PR CI

**AC:** All catalog products green locally; failure message names `system_name` + missing fragment

---

#### Issue 4 — Shared expectation contract (BE + FE)

**Title:** `test: shared seed YAML expectation contract (JSON/YAML, consumed by BE IT + Playwright)`  
**Scope:**
- Move fragment rules from `yaml-expectations.ts` to `testdata/seed/expectations/*.yaml` or single `expectations.yaml`
- Thin TS/Java loaders; `mustContain` / `mustNotContain` / `requiredFiles`
- Eliminate duplicate maintenance

**AC:** Single source; FE E2E and BE IT import same file; TypeScript types generated or hand-maintained

---

#### Issue 5 — Reconcile Playwright E2E with catalog

**Title:** `test: align Playwright yaml-expectations with catalog.yaml (17 products)`  
**Scope:**
- Add expectations for missing catalog products (`rhcl_seed_cors`, `edge_limiting`, …)
- Remove or restore catalog entries for chain products (`claim_role_chain`, `auth_chain`) — **decision required**
- Update `migration-workflow.spec.ts` if product labels change

**AC:** `Object.keys(SEED_YAML_EXPECTATIONS)` matches catalog `system_name` set; manual E2E pass against seeded lab

---

#### Issue 6 — PR CI job: seed catalog parity (no live 3scale)

**Title:** `ci: add seed-catalog-parity job (backend IT, no Admin API)`  
**Scope:**
- New `pr-checks.yml` job or extend `backend-tests-cov` to run `SeedCatalogConversionIT`
- Add row to `docs/STATUS_CHECK_MATRIX.md` when stable
- Optional: fail if `testdata/exports/` older than `catalog.yaml` (mtime/hash check)

**AC:** PR blocks on seed parity failure; runtime &lt; 5 min incremental

---

#### Issue 7 — Nightly lab E2E ( #202 follow-up)

**Title:** `ci: nightly Playwright migration-workflow against seeded 3scale tenant`  
**Scope:**
- `schedule:` workflow with org secrets `THREESCALE_ADMIN_URL`, `THREESCALE_ACCESS_TOKEN`
- Run `npm run test:e2e`; upload trace on failure
- Document in CONTRIBUTING; **do not** add to required PR contexts (per #202)

**AC:** Nightly green on lab; failure artifacts; decision recorded in #202

---

#### Issue 8 — Runtime parity spike (routing class)

**Title:** `test: spike runtime parity — 3scale vs RHCL for routing policies (cors, headers, url_rewriting)`  
**Scope:**
- Kind/OCP lab recipe (doc only or `deploy/` adjunct)
- Apply converted package; HTTP probes via httpbin upstream
- Compare: path routing, CORS headers, header modifiers (no auth)
- Out of PR gate; manual/nightly

**AC:** Design doc + 3 policies proven; go/no-go for auth-class parity

---

#### Issue 9 — Runtime parity (auth class, deferred)

**Title:** `test: runtime parity — auth policies (api_key, anonymous, rate limit, ip_check)`  
**Depends on:** Issue 8 + mock upstream  
**AC:** Documented IdP strategy for OIDC seeds; api_key/anonymous/rate_limit probed

---

#### Issue 10 — Capability profile matrix

**Title:** `test: seed catalog × cluster profile matrix (auto / ocp-4.19 / ocp-4.21)`  
**Scope:** CORS native vs EnvoyFilter, retry, upstream_connection gates per `ConversionOptions`  
**AC:** Parameterized tests for profile variants on subset of seeds (cors, retry, upstream_connection)

### Risks

- **Fixture staleness:** `catalog.yaml` or `3scaleextract` policy defaults change without refreshing exports → false greens or false reds
- **E2E/catalog drift** (already present): chain products in Playwright but not in catalog
- **"Same behavior" scope creep:** Runtime parity for OIDC/JWT/Keycloak needs mock IdP — easy to underestimate
- **#169 performance:** Bulk convert across 17 products in UI E2E may be slow — prefer API-level tests for matrix
- **Windows CRLF:** YAML string tests remain sensitive locally; trust Linux CI

### Ready for Proposal

**Yes** — recommend `/opsx-propose 3scale-pattern-parity-validation` with:
- Locked **P1** scope: Issues 1–4 + 6 (config parity in PR CI)
- **P2** Issues 5 + 7 (lab E2E alignment + nightly)
- **P3** Issues 8–10 (runtime + profile matrix) as separate epics

Orchestrator should ask user:
1. Resolve chain-product drift (add to catalog vs remove from E2E)?
2. Approve epic creation on GitHub with child issues 1–7 first?
