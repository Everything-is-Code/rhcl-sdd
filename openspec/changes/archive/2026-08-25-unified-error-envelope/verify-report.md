# Verify Report — unified-error-envelope

**Change:** unified-error-envelope
**Issue:** [#171](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/171)
**Branch:** `feature/171-unified-error-envelope`
**Date:** 2026-08-25

## Test Results

### Backend — `mvn test`

| Metric | Value |
|--------|-------|
| Tests run | 538 |
| Failures | 0 |
| Errors | 0 |
| Skipped | 0 |
| Result | **BUILD SUCCESS** |

Key test classes: `ApiExceptionMapperTest` (7 tests), `ErrorSanitizerTest` (7 tests), `ArchitectureTest` (21 tests), `ImportControllerTest` (7 tests), `ApplyControllerTest` (16 tests), `ConversionControllerTest` (23 tests).

### Frontend — `npm run typecheck` + `npm test -- --run`

| Metric | Value |
|--------|-------|
| Typecheck | Clean (0 errors) |
| Test files | 17 passed |
| Tests | 62 passed |
| Failures | 0 |

Key test files: `apiError.test.ts` (14 tests — envelope extraction, i18n mapping, legacy fallbacks), `locales.smoke.test.ts` (3 tests — key parity EN/JA).

### Windows CRLF Note

No `ConversionServiceTest` CRLF failures observed. All 124 assertions pass.

## OpenSpec Validate

```
openspec validate unified-error-envelope --type change --store rhcl-sdd --strict
→ Change 'unified-error-envelope' is valid
```

All 4 artifacts complete (proposal, specs, design, tasks). All 18 tasks marked `[x]`.

## Spec Compliance — error-handling

### Req: All HTTP error responses use a unified envelope — PASS

| Scenario | Status | Evidence |
|----------|--------|----------|
| Validation error returns envelope with field details | PASS | `ApiExceptionMapperTest.mapApiException_validationExceptionWithDetails_returns400WithDetails` asserts `error.code=VALIDATION_FAILED` + `error.details.field1` |
| Upstream 3scale failure returns envelope | PASS | `ApiExceptionMapperTest.mapApiException_threeScaleClientException_returns502WithEnvelope` asserts `error.code=THREESCALE_CLIENT_ERROR` + HTTP 502 |
| Uncaught exception returns fallback envelope | PASS | `ApiExceptionMapperResource.mapGenericException()` → 500 `INTERNAL_ERROR`. `WebApplicationException` pass-through confirmed by `PlaywrightE2EIT` (405/404 still work) |

### Req: Error messages never expose secrets — PASS

| Scenario | Status | Evidence |
|----------|--------|----------|
| Token in upstream error is sanitized | PASS | `ErrorSanitizer.sanitize()` regex redacts `access_token`, `token`, `bearer`, `authorization`, `api_key`, `apikey`, `password`, `secret`. `ErrorSanitizerTest` (7 tests) covers key=value, bearer space, and mixed patterns |

### Req: Backend exceptions carry error metadata — PASS

| Scenario | Status | Evidence |
|----------|--------|----------|
| Custom exception maps to correct status and code | PASS | `ApiException` abstract base with `code`/`status`/`details`. 4 subclasses: `ValidationException(400)`, `ThreeScaleClientException(502)`, `ClusterApplyException(500)`, `ImportParseException(400)`. `ApiExceptionMapperTest` covers all 4 |

### Req: Frontend error extraction handles both envelope and legacy shapes — PASS

| Scenario | Status | Evidence |
|----------|--------|----------|
| New envelope shape extracted correctly | PASS | `apiError.test.ts`: "extracts message from new error envelope object" — `{error: {code, message}}` returns `message` |
| Legacy string error still extracted | PASS | `apiError.test.ts`: "returns response.data.error from axios-like error" — string `error` returned |
| Legacy success-false shape still extracted | PASS | `apiError.test.ts`: "handles response.data with success+message shape" — `{success: false, message}` returns `message` |

### Req: ConstraintViolationException produces the unified envelope — PASS

| Scenario | Status | Evidence |
|----------|--------|----------|
| Bean validation failure returns VALIDATION_FAILED envelope | PASS | `ApiExceptionMapperResource.mapConstraintViolation()` produces 400 + `VALIDATION_FAILED` with per-field details. `ApiExceptionMapperTest` covers this (tests the endpoint indirectly via `ExceptionTestResource`). No `@Valid`-annotated controller parameters exist in phase-1 controllers, but the mapper is registered globally |

### Req: Frontend maps error codes to localized messages — PASS

| Scenario | Status | Evidence |
|----------|--------|----------|
| Known error code renders localized message | PASS | `apiErrorI18nMessage()` + `ERROR_CODE_I18N` map in `apiError.ts`. Test: "returns i18n string for known error code" → `VALIDATION_FAILED` → `t('error.validationFailed')`. Keys present in both `en.json` and `ja.json` (6 codes each) |
| Unknown error code falls back to backend message | PASS | Test: "falls back to backend message for unknown error code" → `UNKNOWN_CODE` → returns `error.message` from envelope |
| Legacy error shape still works | PASS | Test: "falls back to legacy extraction for non-envelope errors" → `{error: "string"}` → returns the string, no i18n attempt |

## Spec Compliance — conversion-pipeline (delta)

### Req: REST conversion API is unchanged (MODIFIED) — PASS

| Scenario | Status | Evidence |
|----------|--------|----------|
| Convert endpoint response shape unchanged | PASS | `ConversionControllerTest` (23 tests) — success responses retain original shape. `ConversionServiceTest` (124 tests) — all YAML baselines match |
| Convert endpoint error uses unified envelope | PASS | `ConversionController` throws `ThreeScaleClientException` for global failures. `ConversionControllerTest` asserts 502 + `THREESCALE_CLIENT_ERROR` envelope |
| Per-service partial failure does not use envelope | PASS | Per-service errors remain as typed result items in 200 response. `ConversionControllerTest` covers partial-failure scenario (status in result list, not envelope) |

## Blocking Issues

None.

## Non-Blocking Follow-ups

1. **Phase-2 controller migration**: `ConnectionController`, `SetupController`, `HistoryController`, `ClusterController`, `GatewayInfoController` still use legacy catch blocks — tracked as phase 2 in design.md Non-Goals.
2. **`ConstraintViolationException` live coverage**: No current controller uses `@Valid` on request parameters, so the mapper is tested via `ExceptionTestResource` only. Phase-2 controllers may introduce `@Valid` usage.
3. **Frontend callers not yet using `apiErrorI18nMessage`**: The new i18n-aware helper is available but existing pages still use `apiErrorMessage()`. Adopting `apiErrorI18nMessage` in page-level catch blocks is a separate follow-up.

## Verdict

**PASS** — All 18 tasks complete, all tests green (538 backend + 62 frontend), OpenSpec validate strict passes, all spec scenarios traced to implementation and tests. No blocking gaps.

Recommended next step: `/code-review` then `/opsx-archive`.
