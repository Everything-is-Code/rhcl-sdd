## Adversarial Review

**Scope:** OpenSpec change `unified-error-envelope-complete` / GitHub #196  
**Branch:** `feature/196-unified-error-envelope-complete` (uncommitted working tree)  
**Sources:** `proposal.md`, `design.md`, `specs/error-handling/spec.md`, `tasks.md`; product diff vs `main` + uncommitted changes in `migration-toolkit-rhcl`

**Test runs (reviewer):**
- `cd backend && mvn test` — **PASS**
- `cd frontend && npm test` — **PASS** (17 files, 67 tests)
- `cd frontend && npm run typecheck` — **PASS**

### Spec alignment

- [ ] **All HTTP error responses use unified envelope** — **PARTIAL**  
  Seven targeted controllers throw typed exceptions on failure paths; grep shows no `Map.of("error")` or plain-string `.entity("...")` on controller error paths. **Exception:** `ConnectionController` `@Valid` failures still return Quarkus legacy `{ "violations": [...] }` (see `ConnectionControllerTest` lines 61–87), not `{ error: { code, message, details } }`. Per-service conversion failures on HTTP 200 remain plain `"error"` strings inside `results[]` (in-scope partial-success; messages unsanitized — see findings).

- [x] **Connection test failure (`CONNECTION_TEST_FAILED`, 502)** — **PASS**  
  `ConnectionController` throws `ThreeScaleClientException("CONNECTION_TEST_FAILED", ...)`; `ConnectionControllerTest.testConnection_failure_returns502Envelope` asserts envelope.

- [x] **Resource not found (404 + resource-specific codes)** — **PASS** (implementation)  
  `NotFoundException` used in `HistoryController`, `PackageController`, `GatewayInfoController`, `SettingsController`, `ClusterController`. Mapper test covers `HISTORY_NOT_FOUND`. Several controller tests assert status only, not `error.code` (coverage gap).

- [x] **Missing required parameters → `VALIDATION_FAILED`** — **PASS** for manual throws  
  `ExportController`, `GatewayInfoController`, `PackageController`, `SettingsController`, `HistoryController.deleteByIds` throw `ValidationException`. `ExportControllerTest.getServices_missingParams_returns400` asserts `error.code`.

- [x] **Cluster domain 404 variants** — **PASS** (implementation)  
  `CLUSTER_ROUTE_NOT_FOUND`, `CLUSTER_ROUTE_HOST_PENDING`, `CLUSTER_DOMAIN_EXTRACT_FAILED` in `ClusterController.getDomain()`. **No controller tests** assert envelope codes for these paths.

- [x] **History download failure (`HISTORY_DOWNLOAD_FAILED`, 500)** — **PASS** (implementation)  
  `HistoryController.downloadYaml` catch throws `ClusterApplyException("HISTORY_DOWNLOAD_FAILED", e.getMessage())`. No test for ZIP failure path.

- [x] **SetupController partial success (207)** — **PASS**  
  Unchanged; `SetupControllerTest` still expects 200/207 with `steps[]`, not error envelope.

- [x] **`NotFoundException` in hierarchy** — **PASS**  
  New class + `ApiExceptionMapperTest.mapApiException_notFoundException_returns404WithEnvelope`.

- [ ] **Frontend maps error codes to localized messages (EN + JA)** — **PARTIAL**  
  `ERROR_CODE_I18N` has 14 codes; `en.json` / `ja.json` contain matching `error.*` keys (duplicate top-level `"error"` object — see findings). `ConnectionForm` domain fetch uses `apiErrorI18nMessage` — JA users get localized `CLUSTER_ROUTE_*` / `GATEWAY_*` when envelope is returned.

- [ ] **All production catch blocks use `apiErrorI18nMessage()`** — **PARTIAL**  
  Migrated pages/components per tasks 5.1–5.11; grep shows zero `apiErrorMessage(` outside `apiError.ts` / tests. **`CompatibilityPage` / `compatibilityChecks.ts` still use static `t('compatibility.errorCheck')` or `Error.message`**, not envelope-aware extraction.

- [ ] **Gateway polling: terminal 400/404 vs retry 500/network** — **PARTIAL**  
  `useGatewayUrl` implements `isNonRetryable` for 400/404 only. **502 is retried** (up to ~60s LB phase), then falls through to generic `gwNotReady`. No unit tests for polling behavior.

### Findings

| Severity | Area | Finding | Evidence | Fix (code/spec/tests) |
|----------|------|---------|----------|-------------------------|
| **Major** | Backend / spec | `ConnectionController` bean-validation (`@Valid ConnectionRequest`) responses use legacy `violations` shape, not unified envelope; frontend cannot map `VALIDATION_FAILED` i18n for these failures. | `ConnectionControllerTest` asserts `.body("violations", ...)` (lines 61–87); spec requires envelope for all 4xx validation. `ConversionController` `@Valid` tests assert `error.code` — inconsistent API surface. | Align Connection validation with envelope (custom mapper priority or manual validation); update `ConnectionControllerTest`; verify `ConnectionForm` displays localized validation errors. |
| **Major** | Frontend / polling | `useGatewayUrl` treats **502 as retryable**; after 12×5s retries user sees generic `import.testPanel.gwNotReady`, not `error.gatewayNotFound` or upstream code. Design D8 specifies stop on 400/404 only — 502 retry may be intentional but produces misleading UX. | `useGatewayUrl.ts` `isNonRetryable` checks only 400/404; no test file. | Treat 502 as terminal (or cap retries + surface `apiErrorI18nMessage`); add `useGatewayUrl.test.ts` for 404/400/502/500 paths. |
| **Major** | Tests / tasks | **Tasks claim envelope tests that are missing or weak:** `ClusterControllerTest` has no 404 envelope cases; `HistoryControllerTest` 404/download/delete assert status only (not `error.code`); `PackageControllerTest.downloadFromHistory_notFound` status-only; **no `SettingsControllerTest`** despite task 2.7. | Test files vs `tasks.md` checkmarks. | Add REST-assured assertions for each new code path; add `SettingsControllerTest`. |
| **Major** | Frontend / i18n scope | `compatibilityChecks.ts` catch blocks do not use `apiErrorI18nMessage`; per-service failures always show static `compatibility.errorCheck`. Page-level settings load failure surfaces raw `Error.message`. | `CompatibilityPage.tsx` lines 76–83; `compatibilityChecks.ts` lines 31–37, 61–68, 72–74. | Pass `apiErrorI18nMessage` into `runCompatibilityChecks` deps; migrate catches per spec requirement. |
| **Minor** | i18n / locales | **Duplicate top-level `"error"` key** in `en.json` and `ja.json` (lines ~382 and ~410). JSON parsers keep last entry — new codes work, but first block is dead/confusing. | `frontend/src/locales/en.json`, `ja.json`. | Merge into single `error` object; `locales.smoke.test.ts` does not detect duplicates. |
| **Minor** | Security / partial-success | Bulk `ConversionController` per-service failures (HTTP 200) embed raw `e.getMessage()` in `results[].error` without `ErrorSanitizer`. | `ConversionController.java` lines 158–161. | Sanitize per-item error strings or use stable codes (out of #196 scope but adversarial leak vector). |
| **Minor** | Docs / proposal | Proposal lists `CONVERSION_DNS_REQUIRED`; implementation uses `VALIDATION_FAILED` for `dnsHostname` (spec code list omits `CONVERSION_DNS_REQUIRED`). | `ConversionController` line 77; `spec.md` code registry. | Align proposal/spec or add distinct code. |
| **Question** | Backend | `GatewayInfoController` / `ClusterController` rethrow `RuntimeException` on unexpected K8s errors → `INTERNAL_ERROR` generic message (acceptable per design D6/D8). Is losing upstream detail in logs-only intentional? | `GatewayInfoController.java` lines 83–85; `ClusterController.java` lines 140–142. | Document or wrap in typed exception with sanitized message. |
| **Question** | Frontend | `ConnectionForm.handleFetchDomain` prefixes i18n detail with `connection.domainError:` — JA users get localized code message but English prefix label pattern. | `ConnectionForm.tsx` lines 111–112. | Optional: single i18n key with interpolation. |

### Adversarial checks (requested)

| Check | Result |
|-------|--------|
| Any controller still return plain-string or ad-hoc error JSON on failure paths? | **No** on migrated failure paths. Partial-success 200/207/422 bodies unchanged (`ConversionController` per-item `"error"`, `ApplyController` results). |
| ConnectionForm domain fetch — raw English `CLUSTER_ROUTE_*` when locale `ja`? | **No** — `apiErrorI18nMessage` maps codes to `ja.json` `error.*` keys. |
| `useGatewayUrl`: 502 retry forever? 404 shows `GATEWAY_NOT_FOUND` i18n? | **502 retries** up to LB loop limit then `gwNotReady` (not forever, but ~60s+). **404 stops** and shows localized `error.gatewayNotFound`. |
| `HistoryController` delete empty IDs — envelope vs legacy? | **Envelope** — `ValidationException` → 400; test asserts `error` not null, not legacy plain string. |
| `SetupController` still 207 partial success? | **Yes** — out of scope, unchanged. |
| Secrets in error messages? | **Mapper sanitizes** via `ErrorSanitizer` on `ApiException` paths. `HISTORY_DOWNLOAD_FAILED` passes `e.getMessage()` through sanitizer. Per-service conversion errors on 200 are not sanitized. |
| Missing tests for new `NotFoundException` paths? | **Yes** — mapper test + `GatewayInfoControllerTest`; gaps for cluster/history/package/settings controller 404 paths. |

### Verdict

**PASS WITH GAPS** (post-fix: Major items below addressed in code; remaining gaps are minor/docs)

### Post-fix updates (2026-08-25)

- Connection `@Valid` → manual `ValidationException` with envelope tests.
- Controller envelope tests added (`SettingsControllerTest`, Cluster/History/Export/Package).
- Duplicate locale `"error"` blocks merged.
- `CompatibilityPage` migrated to `apiErrorI18nMessage` via `formatApiError`.
- `useGatewayUrl` shows last envelope error after retry timeout.

Core migration is implemented and green in CI-local runs: typed exceptions on targeted controllers, frontend i18n adoption on listed pages, `NotFoundException` + mapper coverage. Gaps that should be closed before archive: **Connection `@Valid` legacy `violations` response**, **gateway 502 retry UX**, **incomplete controller envelope test assertions**, **`CompatibilityPage` catch blocks outside i18n migration**, and **duplicate locale `error` keys**.

### Before archive

- [ ] Fix or explicitly defer `ConnectionController` `@Valid` envelope mismatch (spec blocker if strict).
- [ ] Add missing controller tests (`ClusterController` 404×3, `HistoryController` envelope codes, `SettingsController`, `PackageController` 404 code).
- [ ] Add `useGatewayUrl` tests; decide 502 terminal vs retry policy and document in `design.md`.
- [ ] Migrate `compatibilityChecks.ts` / `CompatibilityPage` to `apiErrorI18nMessage`.
- [ ] Merge duplicate `error` blocks in `en.json` / `ja.json`.
- [ ] Reconcile `tasks.md` checkmarks with actual test coverage.
