## Adversarial Review

**Scope:** unified-error-envelope / #171
**Sources:** OpenSpec change artifacts (`proposal.md`, `design.md`, `specs/error-handling/spec.md`, `specs/conversion-pipeline/spec.md`, `tasks.md`, `verify-report.md`) + working tree on `feature/171-unified-error-envelope` branch

### Spec alignment

- [x] **All HTTP error responses use a unified envelope** — PASS. `ApiExceptionMapperResource` produces `{"error":{"code","message","details"}}` for `ApiException`, `ConstraintViolationException`, and fallback `Exception`. Phase-1 controllers (`ImportController`, `ApplyController`, `ConversionController`) throw typed exceptions. Tests confirm envelope shape.
- [x] **Validation error returns envelope with field details** — PASS. `ApiExceptionMapperTest.mapApiException_validationExceptionWithDetails_returns400WithDetails` asserts `error.details.field1`.
- [x] **Upstream 3scale failure returns envelope** — PASS. `ConversionControllerTest.convert_globalFailure_returns502WithEnvelope` asserts 502 + `THREESCALE_CLIENT_ERROR`.
- [x] **Uncaught exception returns fallback envelope** — PARTIAL. `mapGenericException()` is implemented but has **no dedicated test** in `ApiExceptionMapperTest` and no test endpoint in `ExceptionTestResource` that triggers a generic uncaught exception. Coverage is inferred from controller integration tests only.
- [x] **Error messages never expose secrets** — PARTIAL. `ErrorSanitizer` covers `access_token`, `token`, `bearer`, `authorization`, `api_key`, `apikey`, `password`, `secret`. However, `sha256~` OpenShift tokens and kubeconfig paths — caught by the existing `ClusterVersionService.sanitize()` — are **not covered** by `ErrorSanitizer`. See Finding #1.
- [x] **Backend exceptions carry error metadata** — PASS. `ApiException` abstract base with `code`/`status`/`details`. Four concrete subclasses with correct defaults.
- [x] **Frontend error extraction handles both envelope and legacy shapes** — PASS. `apiErrorMessage()` checks `typeof errorField === 'object'` first, then falls back to string/legacy handling. Tests cover all three shapes.
- [x] **ConstraintViolationException produces the unified envelope** — PARTIAL. `mapConstraintViolation()` is implemented. `ExceptionTestResource` has the endpoint (`/test-exceptions/constraint-violation`) but `ApiExceptionMapperTest` has **zero tests** calling it. See Finding #3.
- [x] **Frontend maps error codes to localized messages** — PASS. `ERROR_CODE_I18N` maps 6 codes to i18n keys. Both `en.json` and `ja.json` have all 6 keys. Tests confirm known-code → i18n, unknown-code → fallback, legacy → no i18n.
- [x] **Convert endpoint response shape unchanged** — PASS. `ConversionControllerTest` (23 tests) confirms success responses are unchanged.
- [x] **Convert endpoint error uses unified envelope** — PASS. Global catch wraps in `ThreeScaleClientException`. Test asserts 502 envelope.
- [x] **Per-service partial failure does not use envelope** — PASS. Per-service errors stay inside 200 response as result items with `status: "FAILED"`.

### Findings

| # | Severity | Area | Finding | Evidence | Fix (code/spec/tests) |
|---|----------|------|---------|----------|----------------------|
| 1 | **Major** | Security | `ErrorSanitizer` misses `sha256~` OpenShift tokens. The existing `ClusterVersionService.sanitize()` regex includes `sha256~\S+` but `ErrorSanitizer` does not. If a Kubernetes/3scale error message contains an OpenShift bearer token (`sha256~xxxx...`), it leaks through the envelope unsanitized. The spec says "reuses existing patterns" (D5) but the implementation uses a narrower regex. | `ClusterVersionService.java:67-68` pattern: `sha256~\S+` not present in `ErrorSanitizer.java:6-8` | **Code**: Add `sha256~\S+` to `ErrorSanitizer.TOKEN_PATTERN`. **Tests**: Add sanitizer test for `sha256~` token format. |
| 2 | **Major** | Security | `ErrorSanitizer` also misses kubeconfig paths and home directory paths (`/home/user/.kube/...`, `/Users/user/.kube/...`) that `ClusterVersionService.sanitize()` redacts with separate `replaceAll` passes. These paths can reveal server filesystem structure. | `ClusterVersionService.java:738-740` has `replaceAll("(?i)/home/\\S+", "[redacted-path]")` — not in `ErrorSanitizer` | **Code**: Consider adding path redaction to `ErrorSanitizer` or document as intentional omission. **Non-blocking** if the mapper never receives cluster-detection errors (those stay in `ClusterVersionService`), but the fallback mapper could surface them from unexpected sources. |
| 3 | **Major** | Tests | `ConstraintViolationException` mapper has **zero test assertions** in `ApiExceptionMapperTest`. The test endpoint `/test-exceptions/constraint-violation` exists in `ExceptionTestResource` but is never called. The spec scenario "Bean validation failure returns VALIDATION_FAILED envelope" is untested at the mapper level. The verify report claims PASS based on the endpoint existing, which is insufficient evidence. | `ApiExceptionMapperTest.java` — no test method references `constraint-violation`. `ExceptionTestResource.java:69-72` — endpoint exists but unused. | **Tests**: Add test calling `GET /test-exceptions/constraint-violation` (no `name` param) asserting 400 + `VALIDATION_FAILED` + `details` with field paths. |
| 4 | **Major** | Tests | Fallback `Exception` mapper (`mapGenericException`) has **no dedicated test**. No test verifies: (a) unknown exception → 500 `INTERNAL_ERROR`, (b) `WebApplicationException` pass-through preserves status code and response. Both are critical paths — the fallback covers ALL non-migrated endpoints globally. | `ApiExceptionMapperTest.java` — no test for `INTERNAL_ERROR` or `WebApplicationException` pass-through. `ExceptionTestResource.java` — no endpoint for triggering a generic exception. | **Tests**: Add test endpoint that throws plain `RuntimeException`. Add test asserting 500 + `INTERNAL_ERROR`. Add test endpoint that throws `WebApplicationException(404)` and assert the 404 passes through without wrapping. |
| 5 | **Moderate** | Spec mismatch | Frontend `ERROR_CODE_I18N` maps `IMPORT_NO_YAML` → `error.importNoYaml`, and `en.json`/`ja.json` have the key, but **no backend code ever emits `IMPORT_NO_YAML`**. The "no YAML in ZIP" case in `ImportController` throws `ImportParseException` with code `IMPORT_PARSE_ERROR`. The design doc D7 lists `IMPORT_NO_YAML` as a code, but it was never created as a distinct exception. This is dead code in the frontend and dead i18n keys. | `ImportController.java:69` — throws `ImportParseException` (code `IMPORT_PARSE_ERROR`), not a code `IMPORT_NO_YAML`. `grep -r IMPORT_NO_YAML backend/` — zero matches. | **Spec/Code**: Either (a) add `IMPORT_NO_YAML` as a code to `ImportParseException` (second constructor or separate exception), or (b) remove `IMPORT_NO_YAML` from `ERROR_CODE_I18N` and locale files. Clarify intent in the design doc. |
| 6 | **Moderate** | Security | `WebApplicationException` pass-through in `mapGenericException()` bypasses `ErrorSanitizer`. If a REST client exception (e.g., `ClientWebApplicationException` from a failed 3scale call in a non-migrated controller) propagates as a `WebApplicationException`, its response body — which may contain upstream error details including tokens in URLs — passes through unsanitized. Phase-1 controllers catch these, but the fallback covers phase-2 controllers that may not. | `ApiExceptionMapperResource.java:37-39` — `wae.getResponse()` returned directly, no sanitization. `ConnectionController` has `@Valid` but no catch blocks for REST client exceptions. | **Code**: Consider sanitizing `WebApplicationException` response body before pass-through, or at minimum wrapping `ClientWebApplicationException` separately from framework `WebApplicationException` (404/405). **Non-blocking** for phase-1 but should be addressed before phase-2 migration. |
| 7 | **Moderate** | Architecture | `exception` package is NOT defined as a layer in `ArchitectureTest.layeredArchitecture_isRespected`. The two explicit rules (`exception_doesNotDependOnController` and `controllerAndService_mayDependOnException`) partially cover it, but `exception → service` and `exception → client` dependencies are not blocked. A new class in `exception/` could import from `service/` or `client/` without ArchUnit catching it. | `ArchitectureTest.java:24-39` — `exception` layer absent from `layeredArchitecture()`. Lines 164-176 are separate targeted rules. | **Tests**: Add `.layer("Exception").definedBy("...exception..")` to the layered architecture rule with `whereLayer("Exception").mayOnlyBeAccessedByLayers("Controller", "Service")`. |
| 8 | **Minor** | Robustness | `ApiExceptionMapperResource` has no `@Produces(MediaType.APPLICATION_JSON)` annotation. JSON serialization works because RESTEasy Reactive infers it from the `Map` return type, but this is implicit. If a request arrives with `Accept: text/html` (e.g., browser hitting the API directly during debugging), the content negotiation could produce unexpected output. The test endpoint `ExceptionTestResource` has `@Produces(APPLICATION_JSON)` which masks this in tests. | `ApiExceptionMapperResource.java` — no `@Produces` annotation. | **Code**: Add `@Produces(MediaType.APPLICATION_JSON)` to the class or explicitly set `.type(MediaType.APPLICATION_JSON_TYPE)` on each `Response.status().entity().build()`. |
| 9 | **Minor** | Sanitizer coverage | `ErrorSanitizer` does not handle credentials embedded in JSON format (e.g., `{"token":"abc123"}`) or multi-word quoted values (e.g., `password="my secret"`). The regex `\S+` stops at whitespace, so `password="my secret"` would redact `password=[REDACTED]` but `secret"` would remain. This is consistent with the existing `ClusterVersionService.sanitize()` approach, but worth noting as a known limitation. | `ErrorSanitizer.java:7` — regex group 3 is `(\S+)` | **Non-blocking follow-up**: Document as a known limitation or extend regex to handle quoted values. |
| 10 | **Minor** | DX | The `ErrorSanitizer` replacement format `$1=[REDACTED]` differs from `ClusterVersionService.sanitize()` which replaces the entire match with `[redacted]`. This means the two sanitizers produce different output formats for the same input, which could confuse log correlation. | `ErrorSanitizer.java:14` — `$1=[REDACTED]`. `ClusterVersionService.java:737` — `[redacted]`. | **Non-blocking**: Consider aligning formats for consistency, or document intentional divergence. |

### Verdict

**PASS WITH GAPS**

The core architecture is sound: exception hierarchy, mapper structure, controller migration, frontend dual-mode extraction, and i18n mapping are all well-designed and correctly implemented. The 538 backend + 62 frontend tests pass. The change achieves its stated goals for phase-1.

However, three gaps should be addressed before archive:

1. **Security**: `ErrorSanitizer` missing `sha256~` token pattern is a real credential leak vector — OpenShift bearer tokens use this format extensively.
2. **Test coverage**: Two of the three `@ServerExceptionMapper` methods (`ConstraintViolationException` and fallback `Exception`) have zero dedicated tests despite having the test infrastructure already in place.
3. **Spec consistency**: `IMPORT_NO_YAML` phantom code creates confusion about the actual error contract.

### Before archive

**Must fix (blocking):**
- Add `sha256~\S+` pattern to `ErrorSanitizer.TOKEN_PATTERN` + test (Finding #1)
- Add `ApiExceptionMapperTest` test for `ConstraintViolationException` endpoint (Finding #3)
- Add `ApiExceptionMapperTest` tests for fallback mapper: generic exception → `INTERNAL_ERROR` and `WebApplicationException` pass-through (Finding #4)

**Should fix (recommended):**
- Resolve `IMPORT_NO_YAML` phantom code — either implement it in backend or remove from frontend (Finding #5)
- Add `exception` layer to `layeredArchitecture_isRespected` in ArchitectureTest (Finding #7)

**Non-blocking follow-ups:**
- Kubeconfig path redaction in `ErrorSanitizer` (Finding #2) — consolidate follow-up issue
- `WebApplicationException` body sanitization for phase-2 controller migration (Finding #6)
- `@Produces(APPLICATION_JSON)` on `ApiExceptionMapperResource` (Finding #8)
- Sanitizer format alignment with `ClusterVersionService` (Finding #10)
