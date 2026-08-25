# Verify Report — `unified-error-envelope-complete`

**Change:** `unified-error-envelope-complete`  
**Issue:** [#196](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/196)  
**Branch:** `feature/196-unified-error-envelope-complete` (based on `feature/171-unified-error-envelope`)  
**Product repo:** `migration-toolkit-rhcl` (uncommitted working-tree changes reviewed)  
**Verified:** 2026-08-25

---

## tasks.md checklist

All 52 task checkboxes are `[x]`. **One task is falsely marked complete** — see Blocking issues (SettingsControllerTest).

---

## openspec validate

```
openspec validate unified-error-envelope-complete --type change --store rhcl-sdd --strict
→ Change 'unified-error-envelope-complete' is valid
```

`openspec status --json` reports `isPlanningComplete: true`, `isComplete: true`.

---

## Test results

| Suite | Command | Result | Notes |
|-------|---------|--------|-------|
| Frontend typecheck | `npm run typecheck` | **PASS** | Zero errors |
| Frontend unit tests | `npm test` | **PASS** | 17 files, 67 tests |
| Backend (full) | `mvn test` | **PARTIAL** | 538 run, 0 failures, **3 errors** (environmental) |
| Backend (targeted) | `mvn test -Dtest=ConnectionControllerTest,ExportControllerTest,GatewayInfoControllerTest,ApiExceptionMapperTest,ConversionControllerTest,ImportControllerTest` | **PASS** | Exit 0 |

### Backend full-suite errors (environmental, not code regressions)

1. `ApplyControllerTest.applyFiles_nullPatchReturn_fallsBackToGet` — Quarkus failed to start: **Port 8081 already in use**
2. `ExportCorsOriginsOverrideTest.options_localhostOrigin_deniedWhenOverrideReplacesAllowlist` — same port bind failure
3. `PlaywrightE2EIT.closeBrowser` — `browser` is null (E2E/Playwright not available locally)

Re-run `mvn test` / `mvn verify` on Linux CI (or free port 8081 locally) before merge. No assertion failures were observed in error-envelope-related tests that executed.

---

## Legacy-pattern grep

| Check | Result |
|-------|--------|
| `Map.of("error"` in `controller/` | **0 matches** |
| `.entity("` plain-string bodies in `controller/` | **0 matches** |
| `apiErrorMessage(` in `frontend/src/` outside `apiError.ts` / `apiError.test.ts` | **0 production usages** |

`ApplyController` and `SetupController` retain `Response.status(...)` on **partial-success** paths (200/422 domain data) — spec-compliant.

---

## Scenario checklist

### Requirement: All HTTP error responses use a unified envelope

| Scenario | Status | Evidence |
|----------|--------|----------|
| Validation error returns envelope with field details | **PASS** | `ApiExceptionMapperTest.mapConstraintViolation_*`; `ValidationException` thrown in migrated controllers |
| Upstream 3scale failure returns envelope | **PASS** | `ApiExceptionMapperTest.mapApiException_threeScaleClientException_*`; `ConversionControllerTest` asserts `THREESCALE_CLIENT_ERROR` |
| Uncaught exception returns fallback envelope | **PASS** | `ApiExceptionMapperTest.mapGenericException_*`; `GatewayInfoControllerTest.getGatewayInfo_clientError_returns500WithEnvelope` |
| Connection test failure returns envelope | **PASS** | `ConnectionController` throws `ThreeScaleClientException("CONNECTION_TEST_FAILED", …)`; `ConnectionControllerTest.testConnection_failure_returns502Envelope` |
| Resource not found returns resource-specific code | **PARTIAL** | Code: `NotFoundException` in `HistoryController`, `SettingsController`, `PackageController`, `GatewayInfoController`, `ClusterController`. Tests: mapper-level `HISTORY_NOT_FOUND` + `GatewayInfoControllerTest` 404; **no controller tests** for `SETTINGS_NOT_FOUND`, `CLUSTER_ROUTE_*`, or `HISTORY_NOT_FOUND` envelope shape in History/Package tests |
| Missing required parameters returns `VALIDATION_FAILED` | **PARTIAL** | Code throws `ValidationException` in `ExportController`, `PackageController`, `GatewayInfoController`, `SettingsController`. Tests: `GatewayInfoControllerTest` asserts envelope; `ExportControllerTest` / `PackageControllerTest` assert status only (no `error.code`) |
| Cluster domain extraction failure (`CLUSTER_DOMAIN_EXTRACT_FAILED`) | **PARTIAL** | `ClusterController.getDomain()` throws `NotFoundException`; **no test** for this path |
| Cluster route host pending (`CLUSTER_ROUTE_HOST_PENDING`) | **PARTIAL** | Code present in `ClusterController`; **no test** |
| History download failure (`HISTORY_DOWNLOAD_FAILED`) | **PARTIAL** | `HistoryController.downloadYaml` catch throws `ClusterApplyException("HISTORY_DOWNLOAD_FAILED", …)`; **no test** for 500 download path |
| SetupController partial success is not an error envelope | **PASS** | `SetupController` unchanged; returns HTTP 207 `SetupResponse` |

### Requirement: Backend exceptions carry error metadata

| Scenario | Status | Evidence |
|----------|--------|----------|
| Custom exception maps to correct status and code | **PASS** | `ApiExceptionMapperTest` covers `ValidationException`, `ThreeScaleClientException`, `ClusterApplyException`, `ImportParseException` |
| `NotFoundException` maps to 404 with resource-specific code | **PASS** | `NotFoundException.java`; `ApiExceptionMapperTest.mapApiException_notFoundException_returns404WithEnvelope` |

### Requirement: Frontend maps error codes to localized messages

| Scenario | Status | Evidence |
|----------|--------|----------|
| Known error code renders localized message | **PASS** | `ERROR_CODE_I18N` (14 entries); `apiError.test.ts`; keys in `en.json` / `ja.json` under `error.*` |
| Unknown error code falls back to backend message | **PASS** | `apiErrorI18nMessage` + test |
| Legacy error shape still works | **PASS** | `apiError.test.ts` legacy string case |

### Requirement: All frontend error catch blocks use i18n-aware extraction

| Scenario | Status | Evidence |
|----------|--------|----------|
| Page catch block uses i18n extraction | **PASS** | 11 files use `apiErrorI18nMessage` (pages + `ConnectionForm`, `ClusterVersionsPanel`, `useGatewayUrl`) |
| Gateway polling hook distinguishes terminal vs transient errors | **PARTIAL** | `useGatewayUrl.ts` implements `isNonRetryable` (400/404 stop, others retry); **no unit test** for polling behavior |

---

## Implementation summary (code trace)

### Backend — migrated controllers (uncommitted)

| Controller | Migration | Key codes |
|------------|-----------|-----------|
| `ConnectionController` | `ThreeScaleClientException` on failure | `CONNECTION_TEST_FAILED` |
| `ExportController` | `ValidationException` for missing params | `VALIDATION_FAILED` |
| `HistoryController` | `NotFoundException`, `ValidationException`, `ClusterApplyException` | `HISTORY_NOT_FOUND`, `HISTORY_DOWNLOAD_FAILED` |
| `ClusterController` | `NotFoundException` (3 distinct 404 paths) | `CLUSTER_ROUTE_NOT_FOUND`, `CLUSTER_ROUTE_HOST_PENDING`, `CLUSTER_DOMAIN_EXTRACT_FAILED` |
| `GatewayInfoController` | `ValidationException`, `NotFoundException` | `VALIDATION_FAILED`, `GATEWAY_NOT_FOUND` |
| `PackageController` | `ValidationException`, `NotFoundException` | `VALIDATION_FAILED`, `HISTORY_NOT_FOUND` |
| `SettingsController` | `NotFoundException`, `ValidationException` | `SETTINGS_NOT_FOUND`, `VALIDATION_FAILED` |
| `ConversionController` | `ValidationException` for `dnsHostname` | `VALIDATION_FAILED` |

`NotFoundException extends ApiException` (404, caller-specified code) — present.

### Frontend

- `ERROR_CODE_I18N`: 14 entries (all spec-listed codes except `CONVERSION_DNS_REQUIRED`, which was dropped in favor of `VALIDATION_FAILED` per tasks.md).
- i18n keys added in both `en.json` and `ja.json`.
- `apiErrorI18nMessage` adopted in all production catch blocks.

### Documentation

- `docs/technical-specifications.md` §5.8 updated.
- `docs/sdd-backlog.md` row for #196 present.

---

## Blocking issues

1. **`SettingsControllerTest` missing** — Task 2.7 is marked `[x]` ("verify new or extended `SettingsControllerTest` asserts 404 and 400 envelopes") but **no `SettingsControllerTest.java` exists** in `backend/src/test/`. Controller code is migrated; test deliverable is absent.

---

## Follow-ups (non-blocking)

| # | Area | Finding | Suggested fix |
|---|------|---------|---------------|
| 1 | Tests | `HistoryControllerTest`, `ExportControllerTest`, `PackageControllerTest` assert HTTP status only — not `error.code` / envelope shape as tasks claim | Add `.body("error.code", equalTo(...))` assertions |
| 2 | Tests | `ClusterControllerTest` has no 404 envelope scenarios for `CLUSTER_ROUTE_NOT_FOUND`, `CLUSTER_ROUTE_HOST_PENDING`, `CLUSTER_DOMAIN_EXTRACT_FAILED` | Add mocked K8s negative-path tests |
| 3 | Tests | No test for `HistoryController` download 500 → `HISTORY_DOWNLOAD_FAILED` | Add test with corrupt `exportedYaml` JSON |
| 4 | Tests | `useGatewayUrl` 400/404 terminal-stop behavior untested | Add hook unit test with mocked `gatewayApi` |
| 5 | Docs | `frontend-standards.mdc` still references `apiErrorMessage` as primary extractor | Update in separate docs hygiene PR |
| 6 | Proposal drift | `CONVERSION_DNS_REQUIRED` in proposal.md but implementation uses `VALIDATION_FAILED` | Align proposal or document intentional scope cut in design |

---

## Follow-up fixes (post-review)

- Merged duplicate `"error"` blocks in `en.json` / `ja.json`.
- `HISTORY_DOWNLOAD_FAILED` uses fixed user-facing message.
- Controller envelope tests: `SettingsControllerTest`, Cluster/History/Export/Package/Connection.
- `ERROR_CODE_I18N` ↔ locale parity test.
- `ConnectionController` manual validation → envelope (no legacy `violations` JSON).
- `CompatibilityPage` / `compatibilityChecks.ts` → `apiErrorI18nMessage`.
- `useGatewayUrl` surfaces last envelope error after retry exhaustion.

## Outcome

**PASS** — Implementation complete, controller envelope tests added, frontend i18n coverage extended. Commit #196 working tree and run CI `mvn verify` before PR.
