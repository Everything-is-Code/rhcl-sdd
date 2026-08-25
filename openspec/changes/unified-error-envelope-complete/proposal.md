## Why

PR #195 introduced the unified error envelope infrastructure (exception hierarchy, `ApiExceptionMapperResource`, `ErrorSanitizer`) and migrated three controllers (`ImportController`, `ApplyController`, `ConversionController`). Seven controllers still return ad-hoc error shapes (`Map.of("error")`, plain strings, `{ success: false, message }`, bare `404`), and the frontend's `apiErrorI18nMessage()` function exists but is unused — every catch block still calls `apiErrorMessage()`, so users see raw English server messages instead of localized codes. This change completes the migration across the entire stack so #171 can be closed.

## What Changes

- **Backend — remaining controller migration:** `ConnectionController`, `ExportController`, `HistoryController`, `ClusterController`, `GatewayInfoController`, `PackageController`, `SettingsController` throw typed `ApiException` subclasses on every error path; no controller returns `Map.of("error")`, plain-string bodies, or `{ success: false, message }` on failure.
- **New exception class:** `NotFoundException` (404) with per-resource codes (`HISTORY_NOT_FOUND`, `GATEWAY_NOT_FOUND`, `SETTINGS_NOT_FOUND`, `CLUSTER_ROUTE_NOT_FOUND`).
- **New error codes:** `CONNECTION_TEST_FAILED`, `HISTORY_NOT_FOUND`, `HISTORY_DOWNLOAD_FAILED`, `CLUSTER_ROUTE_NOT_FOUND`, `CLUSTER_ROUTE_HOST_PENDING`, `CLUSTER_DOMAIN_EXTRACT_FAILED`, `GATEWAY_NOT_FOUND`, `SETTINGS_NOT_FOUND`, `CONVERSION_DNS_REQUIRED`.
- **Frontend — full i18n adoption:** replace `apiErrorMessage()` with `apiErrorI18nMessage()` in every page/component catch block; extend `ERROR_CODE_I18N` map and `en.json`/`ja.json` for all new codes.
- **Partial-success preserved:** `SetupResponse` (207), `ApplyResult[]` (200), per-service conversion results, and gateway `ready: false` on 200 are domain data, not error responses.
- **`ConversionController` dnsHostname validation** migrated to envelope (`VALIDATION_FAILED` or `CONVERSION_DNS_REQUIRED`).

## Capabilities

### New Capabilities

_(none — building on existing `error-handling` capability)_

### Modified Capabilities

- `error-handling`: Extends the spec with full controller coverage requirement (every controller, not just phase-1), new error codes for 404 resources and connection test, `NotFoundException` in exception hierarchy, and frontend i18n adoption requirement (zero `apiErrorMessage()` in production catch blocks).

## Impact

- **Backend controllers:** `ConnectionController`, `ExportController`, `HistoryController`, `ClusterController`, `GatewayInfoController`, `PackageController`, `SettingsController` (error paths rewritten); `ConversionController`, `ImportController`, `ApplyController` (verify from #195).
- **Backend exception layer:** new `NotFoundException.java`; `ApiExceptionMapperResource` unchanged (existing mappers handle the new subclass).
- **Backend tests:** 10 controller tests updated to assert envelope shape + codes; `ApiExceptionMapperTest` extended for `NotFoundException`.
- **Frontend:** 11 files migrated from `apiErrorMessage()` to `apiErrorI18nMessage()`, `apiError.ts` extended, `apiError.test.ts` extended.
- **i18n:** ~9 new keys in `en.json` and `ja.json`.
- **APIs:** error response bodies change from ad-hoc to envelope; HTTP status codes unchanged. `ConnectionController` success `200 { success: true }` preserved.
- **SDD:** `error-handling/spec.md` delta; `docs/technical-specifications.md` §5.8 finalized.
