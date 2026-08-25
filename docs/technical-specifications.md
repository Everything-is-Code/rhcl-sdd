# Technical Specifications

**Canonical location:** 
hcl-sdd/docs/technical-specifications.md (SDD store). Product code: ../migration-toolkit-rhcl/.
 â€” Migration Toolkit for Red Hat Connectivity Link

Reference document for engineers and coding agents working on this repository. It describes the stack, environment setup, architecture, data model, and conventions **as they actually exist in the code today**, plus a dedicated section of recommended improvements where current practice has gaps.

Companion Cursor rules (loaded automatically when editing matching files):

| Rule | Applies to |
|---|---|
| [`.cursor/rules/data-model.mdc`](../migration-toolkit-rhcl/.cursor/rules/data-model.mdc) | `backend/.../entity/**`, `backend/.../dto/**`, Flyway migrations |
| [`.cursor/rules/testing-standards.mdc`](../migration-toolkit-rhcl/.cursor/rules/testing-standards.mdc) | `backend/src/test/**`, `frontend/src/**/*.test.ts(x)` |
| [`.cursor/rules/frontend-standards.mdc`](../migration-toolkit-rhcl/.cursor/rules/frontend-standards.mdc) | `frontend/src/**/*.ts(x)` |

See also: [`README.md`](../migration-toolkit-rhcl/README.md) (features/workflow/API reference), [`CONTRIBUTING.md`](../migration-toolkit-rhcl/CONTRIBUTING.md) (PR process), [`AGENTS.md`](../migration-toolkit-rhcl/AGENTS.md) (agent-specific context), [`SECURITY.md`](../migration-toolkit-rhcl/SECURITY.md).

---

## 1. Technology stack

| Layer | Technology |
|---|---|
| Backend | Java 21, Quarkus **3.27.5.1**, RESTEasy Reactive (`quarkus-rest-jackson`), Hibernate ORM **Panache**, `quarkus-hibernate-validator`, `quarkus-rest-client-jackson`, SnakeYAML 2.6 |
| Database | PostgreSQL (CrunchyData Operator in-cluster), Flyway migrations (`V1`â€“`V9`) |
| Kubernetes client | Fabric8 Kubernetes Client 6.7.x (`quarkus-kubernetes-client`) |
| API docs | SmallRye OpenAPI + Swagger UI (`/q/swagger-ui`) |
| Frontend | React 18, PatternFly 5 (`@patternfly/react-core/table/icons`), TypeScript 5 (`strict: true`), Vite 4, `react-router-dom` 6, Axios, `i18next` / `react-i18next` |
| Backend testing | JUnit 5 (`quarkus-junit5`), REST-assured, Mockito (`quarkus-junit5-mockito`), `quarkus-panache-mock`, H2 (test profile), **ArchUnit** (layering rules), Playwright (E2E), Awaitility |
| Backend static analysis | Checkstyle, PMD, JaCoCo (merged unit+Quarkus coverage) |
| Frontend testing/lint | Vitest, ESLint 9 (flat config, `typescript-eslint`), `tsc --noEmit` |
| Deployment | OpenShift S2I *or* Helm chart, nginx (frontend static + `/api` reverse proxy), Docker/Podman, GitHub Actions CI/CD, Quay.io image registry |

Full dependency lists: [`backend/pom.xml`](../migration-toolkit-rhcl/backend/pom.xml), [`frontend/package.json`](../migration-toolkit-rhcl/frontend/package.json).

## 2. Development environment setup

| Tool | Version | Purpose |
|---|---|---|
| Java (OpenJDK) | 21+ | Backend build |
| Apache Maven | 3.9.x+ | Backend build/tests |
| Node.js | **22** (matches `frontend/Dockerfile.ci`) | Frontend build |
| npm | 9+ | Frontend deps |
| Docker/Podman | latest | Local image builds (optional) |
| `oc` CLI | matching OCP | `deploy/install.sh` |

```bash
# Backend (PostgreSQL must be reachable on localhost:5432)
cd backend && mvn quarkus:dev

# Frontend (separate terminal)
cd frontend
npm install --legacy-peer-deps
VITE_API_URL=http://localhost:8080 npm run dev
```

- Frontend API base URL is **Vite bake-time** `import.meta.env.VITE_API_URL` (`frontend/src/api/client.ts`) â€” never a runtime `REACT_APP_*` var.
- CORS default allowlist: `http://localhost:5173`, `:3000`, `:8080` â€” override with `CORS_ORIGINS` / `QUARKUS_HTTP_CORS_ORIGINS`.
- Export/API tokens must be sent as `Authorization: Bearer <token>`, never as a query parameter.

Test commands: see [`.cursor/rules/testing-standards.mdc`](../migration-toolkit-rhcl/.cursor/rules/testing-standards.mdc) or the **Testing structure** section below.

## 3. Architecture & file structure

```
                +----------------------+
                |      Web UI          |
                |  (React/PatternFly)  |
                +----------+-----------+
                           |  REST API (JSON)
        +------------------+-------------------+
        |        Quarkus Backend (Java 21)     |
        | Export -> Compatibility -> Convert -> |
        | Validate -> Package -> Apply/Import   |
        | -> History -> Gateway Info -> Setup   |
        +------------------+-------------------+
                    |                  |
              3scale Admin API    PostgreSQL (Flyway)
                    |
        Kuadrant / Gateway API / Istio YAML
```

**Backend package layout** (`backend/src/main/java/com/redhat/migrationtoolkit/rhcl/`, 44 files):

| Package | Files | Role |
|---|---|---|
| `controller/` | 14 | JAX-RS endpoints (HTTP entry point) |
| `service/` | 6 | Business logic (`@ApplicationScoped`) |
| `entity/` | 3 | Panache/JPA entities â€” see [data model](#4-data-model) |
| `dto/` | 7 | Request/response payload shapes |
| `model/` | 11 | Internal domain model (3scale resource graph) |
| `client/` | 1 | REST-client interface to the 3scale Admin API |
| `util/` | 2 | Constants + i18n `ResourceBundle` wrapper |

Layering is enforced by ArchUnit (`backend/src/test/architect/.../ArchitectureTest.java`): `controller â†’ service â†’ client`; controllers must not depend on `client` directly; only `service`/`model` may be `*Service`; entities must not depend on `controller`. See [`testing-standards.mdc`](../migration-toolkit-rhcl/.cursor/rules/testing-standards.mdc) for the full rule list.

**Frontend layout** (`frontend/src/`):

| Folder | Role |
|---|---|
| `components/` | Domain-grouped UI (`import/`, `history/`, `conversion/`, `connection/`, `api/`, `yaml/`) plus `AppStateContext`, `LangSwitcher`, `RouteErrorBoundary` |
| `pages/` | Thin orchestrators per route (<200 lines); own API calls and `useAppState()` writes |
| `api/` | `client.ts` (Axios instance + per-domain functions), `types.ts` (API request/response interfaces only) |
| `locales/` | `en.json` / `ja.json` (react-i18next resources) |
| `styles/` | `pfTokens.ts` (PatternFly CSS-var wrappers), `shared.module.css` |
| `utils/` | Pure helpers (`apiError.ts`, `appStateStorage.ts`, `supportedPolicies.ts`, `clusterCapabilityUi.ts`, `fixHttpRoutePort.ts`, `timezone.ts`) |
| `test/` | Vitest setup (`setup.ts`) |

CSS Modules co-locate with components (e.g. `components/import/import.module.css`). Page-only logic without JSX: `pages/compatibilityChecks.ts`. Full conventions: [`frontend-standards.mdc`](../migration-toolkit-rhcl/.cursor/rules/frontend-standards.mdc).

**API surface**: see [`README.md#api-reference`](../migration-toolkit-rhcl/README.md#api-reference) for the full endpoint table; Swagger UI at `/q/swagger-ui`.

## 4. Data model

Three Panache entities (`ProjectEntity`, `ConversionHistoryEntity`, `AppSettingsEntity`), Flyway-migrated `V1`â€“`V9`. Full field-by-field spec, JSON-column conventions, and suggested improvements (enum columns, `jsonb`): [`.cursor/rules/data-model.mdc`](../migration-toolkit-rhcl/.cursor/rules/data-model.mdc).

Condensed view:

```
Project 1---* ConversionHistory
Project:            id, name, threescaleUrl, tenant, createdAt, updatedAt
ConversionHistory:  id, project_id, source(CONVERT|IMPORT), serviceId, serviceName,
                     namespace, status(COMPLETED|PARTIAL|FAILED|IN_PROGRESS),
                     compatibilityScore, totalCount, successCount, failureCount,
                     failureDetails(JSON text), exportedYaml(JSON text), yamlContent,
                     packageName, createdAt
AppSettings:         settings_key (PK), value (TEXT), updatedAt   -- generic k/v store
```

## 5. Design patterns and conventions

### 5.1 API design patterns (backend)

- REST/JAX-RS via RESTEasy Reactive: `@Path("/api/...")`, `@Produces/@Consumes(APPLICATION_JSON)`, MicroProfile OpenAPI annotations (`@Tag`, `@Operation`) â€” **no `@APIResponse`** used anywhere yet.
- Dependency injection: **field injection** with `@Inject` everywhere (no constructor injection observed) â€” follow the existing pattern for consistency rather than introducing constructor injection ad hoc.
- Request validation: `@Valid` on the DTO parameter (`ConversionRequest`, `ConnectionRequest`) triggers automatic 400s from Bean Validation; additional business-rule checks are inline `if` statements returning `Response.status(400).entity(Map.of("error", ...))`.
- Response shapes: mostly typed DTOs (`ClusterVersionsResponse`, `ValidationResult`, `ServiceListPage`) or **records nested inside the controller itself** for endpoint-specific shapes (`ApplyController.ApplyResult`, `SetupController.SetupResponse`) â€” no shared response envelope.
- `@Transactional` is applied at the **controller** method level (not inside services) wherever Panache entities are persisted.
- File upload uses RESTEasy Reactive `@RestForm("file") FileUpload` (multipart) â€” see `ImportController`.

### 5.2 Testing structure

Full spec (ArchUnit rules, naming, frameworks, Vitest conventions): [`.cursor/rules/testing-standards.mdc`](../migration-toolkit-rhcl/.cursor/rules/testing-standards.mdc). Summary: JUnit5 + REST-assured + Mockito + H2 for backend, ArchUnit for layering, Playwright for E2E; Vitest for frontend utils/API mocks + minimal RTL smoke (`*.test.tsx` with jsdom). Full component coverage tracked in [#172](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/172).

### 5.3 Naming conventions

| Scope | Convention |
|---|---|
| Backend packages | `com.redhat.migrationtoolkit.rhcl.{controller,service,entity,dto,model,client,util}`, singular, lowercase |
| Backend classes | Suffix by role: `*Controller`, `*Service`, `*Entity`, `*Client`; DTOs have **no** `Dto` suffix (`ConversionRequest`, not `ConversionRequestDto`) |
| Backend constants | `UPPER_SNAKE_CASE`, grouped in a `*Constants` class with a private constructor |
| Backend logger | Always `private static final Logger LOG = Logger.getLogger(<Class>.class);` |
| Backend test methods | `methodUnderTest_condition_expectedResult` |
| Frontend files | PascalCase for page/component files, camelCase for non-component utils |
| Frontend types | `interface` for object shapes, `type` for string-literal unions; `Request`/`Response`/`Result`/`Page` suffixes for API payloads |
| Frontend handlers | `handleX` inside a component, `onX` for a prop passed to a child |
| Frontend booleans | `isX` or a participle (`loading`, `applying`) |
| i18n keys | `{page}.{camelCaseKey}`, prefixes `btnX`/`labelX`/`colX`/`errorX`/`ariaX` |

### 5.4 Git workflow

- Default branch: `main`. Branch from `main` per issue/feature: `feature/<issue>-short-description` (see [development_guide.md](./development_guide.md)).
- **Conventional Commits**: `feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:` â€” see [`CONTRIBUTING.md`](../migration-toolkit-rhcl/CONTRIBUTING.md).
- Keep PRs focused; prefer review slices under ~400 authored lines. Link issues (`Closes #â€¦`).
- Review required from [`.github/CODEOWNERS`](.github/CODEOWNERS) (`@pcastelo`, `@fmenesesg`). CI must be green before merge.
- **Stacked/dependent PRs**: merge the root PR into `main` first (keep its branch), retarget the dependent PR's base to `main`, then resolve the resulting conflict by merging `main` in â€” don't rebase a PR that's already open for review. Historical conflict hotspots were `ConversionService.java`; after #40 the orchestrator is thin (~175 lines) â€” new policy work should touch generators/contributors instead.
- Every PR gets an AI code-review comment (standard disclaimer, English, findings ranked Major/Moderate/Minor/Nit, never auto-approve unless asked).

### 5.5 Clean Code / SOLID

- Controllers/services are generally single-purpose and thin. `ConversionService` is now a thin orchestrator (~175 lines) that builds `ConversionContext` and invokes `ResourceGeneratorRegistry`; YAML generation lives in `service/generator/` and `service/generator/contributor/` ([#40](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/40), branch `feature/conversion-strategy-registry`, pending merge).
- The rest of the backend follows the layering ArchUnit enforces (dependency inversion between layers is structurally guaranteed, not just convention).
- Frontend pages are orchestrators under `pages/` with subcomponents in `components/<domain>/` ([#41](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/41), PR [#191](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/191) pending merge). `AppState` lives in `AppStateContext`; shared errors in `utils/apiError.ts`.

### 5.6 Design patterns â€” current and planned

| Pattern | Where |
|---|---|
| Repository (implicit) | Panache active-record entities (`ProjectEntity.findLatestByServiceId`, etc.) |
| Cache-aside (manual) | `ThreeScaleExportService`'s `ConcurrentHashMap` + TTL caches (`exportCache`, `backendCatalogCache`, `applicationsCache`); keys include a SHA-256 token fingerprint to prevent cross-tenant cache leaks |
| Request coalescing | `ClusterVersionService`'s `ConcurrentHashMap<String, CompletableFuture<...>>` collapses concurrent detects into one in-flight probe |
| Error boundary | `components/RouteErrorBoundary.tsx` wraps `<Routes>` in `App.tsx` |
| React Context | `AppStateContext` + `useAppState()` for workflow state (replaces prop drilling) |
| Strategy + Registry | `ResourceGenerator` per output file + `ResourceGeneratorRegistry` (CDI); orchestrator delegates via `convert()` â€” [#40](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/40) |
| Collector/Contributor | `HttpRouteContributor`, `AuthPolicyContributor`, `SecretContributor` against shared builders for multi-policy YAML files â€” [#40](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/40) |
| Readme notes collector | `ReadmeSupport.build(..., ReadmeNotes)` replaces growing `buildReadme` positional args â€” overlaps [#170](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/170) |

### 5.7 Form validation patterns

Backend: Bean Validation (`@NotBlank`/`@NotEmpty`) on request DTOs via `@Valid`, plus inline manual `if` checks for cross-field business rules (e.g. `dnsHostname` required when `includeDnsPolicy=true`).

Frontend: no schema library. Manual `if` + boolean state, surfaced via PatternFly `FormGroup`/`FormHelperText` (preferred, e.g. `ConnectionPage`) or a plain `<span>` with a CSS-module error class (e.g. `ImportPage`). Submit buttons are disabled rather than showing a submit-time error when a required field is missing. Full detail: [`frontend-standards.mdc`](../migration-toolkit-rhcl/.cursor/rules/frontend-standards.mdc).

### 5.8 Error handling

- **Backend**: no custom exception classes, no `ExceptionMapper`/`@ServerExceptionMapper` anywhere in the codebase. The pattern is `try { ... } catch (Exception e) { LOG.warnf(e, ...); return Response.status(...).entity(Map.of("error", ...)).build(); }` repeated per endpoint, with an **inconsistent error body shape** across controllers (`{"error": "..."}` vs `{"success": false, "message": "..."}` vs typed result records).
- **Frontend**: each page extracts `e.response?.data?.error || e.response?.data?.message || e.message || fallback` and renders it in an `Alert variant="danger"` â€” this extraction logic is duplicated per page rather than shared.

### 5.9 Logging

- Backend: exclusively `org.jboss.logging.Logger`, always named `LOG`, always with placeholder methods (`LOG.infof(...)`, `LOG.warnf(...)`, `LOG.debugf(...)`) â€” never string concatenation, never `System.out`. Enforced by ArchUnit. Levels: `INFO` for milestones, `WARN` for recoverable/soft-fail paths, `DEBUG` for low-level probe detail.
- Secrets are redacted before logging: `ClusterVersionService.sanitize()` strips tokens/`bearer`/`sha256~`/kubeconfig paths from any message that might reach a log or an error response.
- Frontend: `console.error` is used **only** inside the two error boundaries (`App.tsx`, `ImportPage.tsx`) â€” no other console logging, no client-side log shipping.

### 5.10 Cybersecurity practices

- Tokens travel as `Authorization: Bearer <token>` only â€” query-string tokens intentionally unsupported.
- Cache keys include a SHA-256 fingerprint of the access token (`ThreeScaleExportService.tokenFingerprint()`) so two tenants/users can never share cached data.
- `ConversionConstants.CREDENTIAL_PLACEHOLDER = "REPLACE_ME"` is written into generated `secret.yaml` instead of real credentials; `ValidationService` flags this placeholder if left unreplaced before apply.
- RBAC least-privilege: `ApplyController` provisions a `Role`/`RoleBinding` (or `ClusterRole` when needed) scoped to the exact `apiGroup`s/verbs required (Istio, Gateway API, Kuadrant, core), rather than a broad admin binding.
- Frontend never persists the access token: `sessionStorage` persistence (`appStateStorage.ts`) explicitly strips `accessToken`/`connected` before saving.
- CORS is allowlist-based, overridable via env var, never `*`.
- **Gap**: the backend itself has no authentication/authorization layer of its own â€” it trusts whatever 3scale/cluster credentials the caller supplies per request. Acceptable for a local/admin tool, but relevant if this is ever exposed beyond a trusted network (see suggestions below).

### 5.11 Code documentation & comments

- Javadoc/JSDoc is sparse and used only where a field's meaning is non-obvious (e.g. `ServiceListPage.total` nullability), not on every method.
- Prefer comments that explain **why** (invariants, security constraints â€” e.g. the `I-2`/`I-3` markers in `appStateStorage.ts` referencing the "never persist the token" invariant) over comments that restate the code.
- i18n message keys with inline default text (`t('key', 'Default text')`) double as lightweight documentation of a string's purpose when the translation doesn't exist yet.
- No repo-wide Javadoc/TSDoc generation is configured â€” don't add one without a maintainer decision, since none of the current docs pipeline depends on it.

## 6. Applicability across backend / frontend / mobile

These conventions are written to apply uniformly across every client surface of this program:

- **Backend**: sections 5.1â€“5.11 above apply directly; entity/testing detail lives in [`data-model.mdc`](../migration-toolkit-rhcl/.cursor/rules/data-model.mdc) and [`testing-standards.mdc`](../migration-toolkit-rhcl/.cursor/rules/testing-standards.mdc).
- **Frontend (current, only client today)**: full detail in [`frontend-standards.mdc`](../migration-toolkit-rhcl/.cursor/rules/frontend-standards.mdc) â€” component structure, API client, form validation, error handling, state, i18n, styling, naming.
- **Mobile**: **not implemented today** â€” there is no native or React Native client. If one is built, it should reuse `frontend/src/api/types.ts` contracts, the same error-message-extraction convention, and the same i18n key structure/locale files documented in `frontend-standards.mdc`, rather than duplicating them.

## 7. Suggestions / recommended improvements

Gaps identified while writing this document, prioritized. Items already tracked as GitHub issues are linked; the rest are new observations from this review.

| # | Area | Suggestion | Tracking |
|---|---|---|---|
| 1 | Backend architecture | ~~Split `ConversionService`~~ **Done on branch** `feature/conversion-strategy-registry` — merge PR to close [#40](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/40) | [#40](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/40) |
| 2 | Backend error handling | Introduce a `@ServerExceptionMapper`/custom exception hierarchy and a **single** error-response envelope (`{error, code, details}`) instead of 3 inconsistent ad-hoc shapes across controllers | New â€” not yet tracked |
| 3 | Backend performance | Parallelize bulk convert, fix N+1 3scale calls, expose backend pagination in `HistoryPage` | [#169](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/169) |
| 4 | Policy coverage | Implement the 19 recognized-but-unconverted 3scale policies | [#149](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/149) (+ 19 child issues) |
| 5 | `generateReadme` signature | ~~Positional note args~~ **Addressed** via `ReadmeSupport` + `ReadmeNotes` collector on #40 branch — confirm closure of [#170](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/170) on merge | [#170](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/170) |
| 6 | Frontend testing | ~~Add `@testing-library/react` + `jsdom`~~ **Partial on #41 branch** — utils + `AppStateContext` + smoke RTL; full coverage in [#172](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/172) | [#172](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/172) |
| 7 | Frontend error handling | ~~Extract `apiErrorMessage`~~ **Done on #41 branch** (`frontend/src/utils/apiError.ts`) — merge PR [#191](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/191) | [#41](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/41) |
| 8 | Data model | Use `@Enumerated(EnumType.STRING)` for `status`/`source` instead of free-form `String`; consider native `jsonb` for `failureDetails`/`exportedYaml` | New â€” not yet tracked |
| 9 | Backend security | Consider whether the tool ever runs outside a trusted network; if so, add an auth layer in front of the backend's own API (today it has none â€” it only proxies caller-supplied 3scale/cluster credentials) | New â€” not yet tracked |
| 10 | Form validation | Once a 3rd/4th frontend form repeats the same manual-`if` pattern, extract a small shared validation helper rather than duplicating it again | New â€” not yet tracked |
| 11 | Documentation | ~~Revisit after #40~~ **Updated** for #40 + #41 frontend split (2026-08-25); revisit on error-handling envelope change (#2) | Partially addressed |

---

*Generated from audits of `backend/src` and `frontend/src`. Last major frontend update: #41 (`frontend-component-split`, PR #191 pending). Re-run after #169 / backend envelope work.*

