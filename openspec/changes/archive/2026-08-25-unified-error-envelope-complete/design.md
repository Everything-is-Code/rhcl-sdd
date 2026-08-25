## Context

See proposal.md — Why. PR #195 delivered the exception infrastructure (`ApiException` hierarchy, `ApiExceptionMapperResource`, `ErrorSanitizer`) and migrated three controllers. The mapper already catches all uncaught exceptions (fallback), `ConstraintViolationException`, and every `ApiException` subclass — so the infrastructure is stable and this change only adds a new subclass and rewrites controller error paths.

Current state (on `feature/171-unified-error-envelope`):
- Backend: `exception/` package with `ApiException`, `ValidationException`, `ThreeScaleClientException`, `ClusterApplyException`, `ImportParseException`, `ErrorSanitizer`, `ApiExceptionMapperResource`.
- Frontend: `apiErrorMessage()` (dual-mode: envelope object + legacy), `apiErrorI18nMessage()` (unused), `ERROR_CODE_I18N` with 6 codes.
- Seven controllers still build ad-hoc error responses.

## Goals / Non-Goals

**Goals:**
- Every controller error path uses typed exceptions; no manual `Response.status().entity(Map.of(...))` on failure.
- Frontend uses `apiErrorI18nMessage()` in every catch block with full EN/JA coverage.
- Single PR, single branch, closes #196 and #171.

**Non-Goals:**
- Changing success response shapes (pagination envelope, `ApplyResult[]`, `SetupResponse`, gateway `ready` flags on 200).
- Migrating `StepResult.message` strings to i18n (backend partial-success messages remain English).
- Removing `apiErrorMessage()` from the module (stays as internal fallback).
- Touching `ValidationController` or `DefaultsController` (no error paths).

## Decisions

### D1 — Reuse existing infrastructure, add only `NotFoundException`

**Choice:** Add a single `NotFoundException extends ApiException` (status 404) with a caller-specified code. All other exceptions (`ValidationException`, `ThreeScaleClientException`, `ClusterApplyException`) are reused as-is.

**Why:** The mapper already handles any `ApiException` subclass generically. A dedicated 404 class makes intent clear and avoids overloading `ValidationException` for not-found semantics. One class covers `HISTORY_NOT_FOUND`, `GATEWAY_NOT_FOUND`, `SETTINGS_NOT_FOUND`, `CLUSTER_ROUTE_NOT_FOUND` via the code parameter.

**Alternative considered:** Per-resource exception classes (`HistoryNotFoundException`, etc.) — rejected as unnecessary granularity; the `code` field already distinguishes them.

### D2 — ConnectionController: throw on failure, keep `{ success: true }` on success

**Choice:** On failed 3scale test, throw `ThreeScaleClientException` with code `CONNECTION_TEST_FAILED` (status 502). On success, keep `Response.ok(Map.of("success", true, ...))`.

**Why:** Axios already enters the `catch` path on 502, so the frontend flow is unchanged. The success `{ success: true }` body is consumed by `ConnectionForm.tsx` and changing it would break the flow needlessly. The envelope is only for errors.

**Alternative considered:** New `ConnectionException` — rejected because `ThreeScaleClientException` already carries the right status (502) and the code parameter distinguishes it.

### D3 — Granular 404 codes vs. generic `NOT_FOUND`

**Choice:** Use resource-specific codes (`HISTORY_NOT_FOUND`, `GATEWAY_NOT_FOUND`, `SETTINGS_NOT_FOUND`, `CLUSTER_ROUTE_NOT_FOUND`) instead of a single `NOT_FOUND`.

**Why:** The frontend can show different localized messages per context (e.g. "History entry not found" vs. "Gateway not found in cluster"). The overhead is one i18n key per code — trivial. It also makes API documentation and debugging clearer.

### D4 — Cluster domain: three distinct 404 codes

**Choice:** `ClusterController.getDomain()` currently has three distinct 404 paths:
1. Route not found → `CLUSTER_ROUTE_NOT_FOUND`
2. Route exists but no host → `CLUSTER_ROUTE_HOST_PENDING`
3. Route host doesn't contain `.apps.` → `CLUSTER_DOMAIN_EXTRACT_FAILED`

Keep all three as distinct codes.

**Why:** These represent different operational states that require different user actions (deploy route vs. wait for DNS vs. investigate route config). A single `NOT_FOUND` would lose actionable context.

### D5 — Frontend migration pattern

**Choice:** Replace `apiErrorMessage(e, fallback)` with `apiErrorI18nMessage(e, t, fallback)` in every catch block. The `t` function is already available via `useTranslation()` in all page components.

Pattern:
```tsx
catch (e: unknown) {
  setError(apiErrorI18nMessage(e, t, t('page.fallbackError')));
}
```

For pages that currently wrap the message in `t('page.errorKey', { message: apiErrorMessage(...) })`, simplify to just `apiErrorI18nMessage(e, t)` when the error code's i18n key already provides a good message. Keep the wrapper pattern only when the page needs to prepend context (e.g. "Conversion failed: {message}").

**Why:** Keeps the migration mechanical and minimizes page-level logic changes.

### D6 — ExportController: validation exceptions, let service layer propagate 3scale errors

**Choice:** Replace the three plain-string 400 responses with `throw new ValidationException("url query parameter and Authorization Bearer token are required")`. For upstream 3scale failures in `ThreeScaleExportService`, let them propagate as `ThreeScaleClientException` (or uncaught → mapper fallback with `INTERNAL_ERROR`).

**Why:** The service layer already throws on HTTP failures. The mapper catches all uncaught exceptions. No new try/catch needed in ExportController — just remove the existing parameter validation `if` blocks and throw instead of `return Response.status(400)`.

### D7 — SetupController: no envelope changes

**Choice:** Keep `SetupController` exactly as-is. It has no global error paths — partial failures are per-step `StepResult` entries in a 207 response, which are domain data per the spec.

**Why:** Adding envelope errors to per-step results would break the `SetupResponse` contract and the frontend that reads `steps[].success`. If a truly unexpected exception occurs, the fallback mapper already covers it.

### D8 — `useGatewayUrl` terminal vs. transient error distinction

**Choice:** In the gateway polling hook, catch errors and check status: 400/404 → stop polling, show `apiErrorI18nMessage()`; 500/network → continue retry loop; timeout → existing `t('import.testPanel.gwNotReady')`.

**Why:** A 400 (bad params) or 404 (gateway doesn't exist) won't resolve on retry. A 500 or transient network error may resolve as the cluster stabilizes.

## Risks / Trade-offs

**[Risk] PR #195 not merged yet** → This change depends on the exception infrastructure from #195. Mitigation: branch from `feature/171-unified-error-envelope` or rebase onto `main` after #195 merges.

**[Risk] Controller tests assert legacy error shapes** → Existing tests check for `Map.of("error")` or plain strings. Mitigation: update assertions to check envelope shape (`error.code`, `error.message`) in the same commit that migrates each controller. Status codes are unchanged.

**[Risk] Frontend tests mock `apiErrorMessage`** → Mitigation: update mocks to use `apiErrorI18nMessage`; add mock `t()` function in test setup.

**[Trade-off] ConnectionController `{ success: true }` asymmetry** → Success returns a non-envelope body while failure uses the envelope. Acceptable: the envelope is for errors only per spec; the success body is consumed by existing frontend logic.

**[Trade-off] SettingsController 404 on GET may not surface to user** → `loadSupportedPolicies()` silently falls back to defaults on 404. The envelope change doesn't affect this because the frontend catches and ignores the error. Document but don't change.
