## 1. Exception hierarchy and mapper

- [x] 1.1 Create `ApiException` abstract base class in `exception/` package with `code` (String), `status` (int), and `details` (Map) fields. Verify: class compiles and `mvn compile` succeeds.
- [x] 1.2 Create `ValidationException` (400, `VALIDATION_FAILED`), `ThreeScaleClientException` (502, `THREESCALE_CLIENT_ERROR`), `ClusterApplyException` (500, `APPLY_FAILED`), and `ImportParseException` (400, `IMPORT_PARSE_ERROR`) extending `ApiException`. Verify: each subclass instantiates with correct defaults in a unit test.
- [x] 1.3 Add `sanitize(String)` utility method to `ApiException` (or a shared `ErrorSanitizer` class) that redacts access tokens and credentials from messages, reusing the pattern from `ClusterVersionService.sanitize()`. Verify: unit test confirms tokens like `access_token=abc123` are replaced with `[REDACTED]`.
- [x] 1.4 Create `ApiExceptionMapperResource` CDI bean with three `@ServerExceptionMapper` methods: one for `ApiException` → envelope with `exception.getStatus()`, one for `ConstraintViolationException` → 400 `VALIDATION_FAILED` with field details, and one for `Exception` (fallback) → 500 `INTERNAL_ERROR` with generic message. All call `sanitize()` on the message. Verify: `ApiExceptionMapperTest` with `@QuarkusTest` confirms each mapper produces correct HTTP status + envelope JSON shape.

## 2. ArchUnit layering

- [x] 2.1 Update `ArchitectureTest.java` to include the `exception` package in layering rules: `exception` must not depend on `controller`; `controller` and `service` may depend on `exception`. Verify: `mvn test -Dtest=ArchitectureTest` passes.

## 3. Controller migration — ImportController

- [x] 3.1 Replace `ImportController` catch blocks with typed exception throws: `fileUpload == null` → `ValidationException`, `IOException` parsing ZIP → `ImportParseException`, no YAML in ZIP → `ImportParseException`. Remove manual `Response.status(...).entity(Map.of(...))` construction. Verify: existing `ImportController` tests pass with envelope shape assertions (status codes unchanged).
- [x] 3.2 Add test cases for `ImportController` error scenarios asserting the envelope JSON structure (`error.code`, `error.message` present, correct HTTP status). Verify: `mvn test -Dtest=ImportControllerTest` passes.

## 4. Controller migration — ApplyController

- [x] 4.1 Replace `ApplyController` top-level catch with `ClusterApplyException` for infrastructure failures. Keep `ApplyResult[]` partial-success pattern unchanged (200 + list). Replace `files == null/empty` check with `ValidationException`. Verify: existing apply tests pass; partial-success response is still 200 with `ApplyResult` list.
- [x] 4.2 Add test cases for `ApplyController` envelope responses: empty files → 400 `VALIDATION_FAILED`, infra failure → 500 `APPLY_FAILED`, partial file failure → still 200 `ApplyResult`. Verify: `mvn test -Dtest=ApplyControllerTest` passes.

## 5. Controller migration — ConversionController

- [x] 5.1 Replace `ConversionController` global catch (outside the per-service loop) with `ThreeScaleClientException` for upstream failures. Per-service partial failures inside the loop remain as typed result items (not envelope). Verify: existing `ConversionServiceTest` and controller tests pass.
- [x] 5.2 Add test case for `ConversionController` global failure asserting the envelope (`THREESCALE_CLIENT_ERROR`, 502). Verify: `mvn test` passes.

## 6. Frontend error extraction and i18n mapping

- [x] 6.1 Extend `apiErrorMessage()` in `frontend/src/utils/apiError.ts` to detect when `error` field is an object (`typeof error === 'object' && error.message`) and extract `error.message`. Legacy string/success-false fallbacks remain. Verify: `npm run typecheck` clean.
- [x] 6.2 Add an `ERROR_CODE_I18N` map (`Record<string, string>`) in `apiError.ts` mapping phase-1 codes to i18n keys: `VALIDATION_FAILED` → `error.validationFailed`, `THREESCALE_CLIENT_ERROR` → `error.threescaleClient`, `APPLY_FAILED` → `error.applyFailed`, `IMPORT_PARSE_ERROR` → `error.importParse`, `IMPORT_NO_YAML` → `error.importNoYaml`, `INTERNAL_ERROR` → `error.internal`. Add a new `apiErrorI18nMessage(e: unknown, t: TFunction, fallback?: string)` helper that uses the map when the code is recognized, falling back to `apiErrorMessage()` otherwise. Verify: `npm run typecheck` clean.
- [x] 6.3 Add i18n keys for all phase-1 error codes in `frontend/src/locales/en.json` and `frontend/src/locales/ja.json`. EN values should be user-friendly messages (e.g. `"Validation failed. Please check your input."`). JA values should be accurate Japanese translations. Verify: `locales.smoke.test.ts` still passes.
- [x] 6.4 Add test cases in `apiError.test.ts`: (a) envelope with known code + mock `t` → returns localized string, (b) envelope with unknown code → falls back to backend `message`, (c) legacy shape → i18n mapping not attempted. Verify: `npm test -- --run` all green including existing 10+ cases.

## 7. Integration verification

- [x] 7.1 Run full backend test suite: `cd backend; mvn test`. All tests green including new mapper tests, controller envelope assertions, and ArchUnit rules.
- [x] 7.2 Run full frontend test suite: `cd frontend; npm run typecheck; npm test -- --run`. All tests green including new envelope extraction and i18n mapping tests.
- [x] 7.3 Update `docs/technical-specifications.md` §5.8 in the SDD store to describe the unified error envelope pattern, exception hierarchy, i18n code mapping, and phase-1 controller coverage. Verify: file updated with accurate content.
