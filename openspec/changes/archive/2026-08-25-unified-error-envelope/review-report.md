## AI Code Review

_Disclaimer: AI-generated review. Human maintainer review required._

**Change:** unified-error-envelope | **Issue:** [#171](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/171)

---

### Major

**M1. `ErrorSanitizer` bypass on `Authorization: Bearer <token>` pattern (security)**

The regex captures keyword → separator → single non-whitespace token. For input `Authorization: Bearer eyJhbGci...`, the first match consumes `(Authorization)(: )(Bearer)` and replaces it with `Authorization=[REDACTED]`, but the actual JWT `eyJhbGci...` following `Bearer` is **not captured** and leaks verbatim into the envelope `message`. The test `sanitize_bearerToken_redacted` only covers `bearer xyz789token` (single word following `bearer`), not the two-word `Authorization: Bearer <value>` form that HTTP clients may embed in exception messages.

**Spec ref:** "Error messages never expose secrets" — `THREESCALE_CLIENT_ERROR` messages could contain header dumps from upstream HTTP failures.

**Fix location:** `ErrorSanitizer.java` — either run the regex in a loop until no match, or add a second pass that catches `Bearer\s+\S+` patterns independently. Add a test case: `sanitize("Authorization: Bearer eyJhbGci.secret")` must not contain `eyJhbGci`.

---

**M2. `ApiExceptionMapperTest` missing tests for 2 of 3 mapper methods**

The test covers all 4 `ApiException` subclasses (7 tests) but has **zero tests** for:

- **`mapGenericException`** — the fallback `Exception → INTERNAL_ERROR (500)` mapper. This is a spec-mandated scenario ("Uncaught exception returns fallback envelope"). The `ExceptionTestResource` has no endpoint that throws a generic non-ApiException.
- **`mapConstraintViolation`** — the `ConstraintViolationException → VALIDATION_FAILED` mapper with per-field details. The `ExceptionTestResource` has a `/constraint-violation` endpoint but no test in `ApiExceptionMapperTest` exercises it.

**Spec ref:** Both scenarios are explicit requirements in `specs/error-handling/spec.md` (scenarios 3 and 6).

**Fix location:** Add 2-3 tests to `ApiExceptionMapperTest` and add a `/generic-error` endpoint to `ExceptionTestResource`.

---

### Moderate

**Mo1. `IMPORT_NO_YAML` — dead i18n mapping (spec-vs-implementation mismatch)**

`ERROR_CODE_I18N` and both locale files map code `IMPORT_NO_YAML`, and design D7 lists it as a phase-1 code. But no backend exception produces this code. `ImportController` throws `ImportParseException` (code: `IMPORT_PARSE_ERROR`) for both "corrupt ZIP" and "no YAML files found" cases.

The user receives the same localized error message for fundamentally different problems (file corruption vs. missing YAML content). Either `ImportController` should use a separate `IMPORT_NO_YAML` code (via a new subclass or a code override), or the frontend mapping is premature and should be removed to avoid confusion.

---

### Minor

**m1. `WebApplicationException` pass-through is untested**

`mapGenericException` passes through `WebApplicationException` to preserve JAX-RS standard 404/405 responses. This is intentional and correct, but has no dedicated test. A future change could accidentally wrap these in the error envelope, breaking standard JAX-RS routing behavior.

**Fix location:** Add a test in `ApiExceptionMapperTest` that issues a request to an undefined path and asserts the original 404 response is preserved (not wrapped in envelope).

---

**m2. `ConstraintViolation` duplicate-property collision**

`mapConstraintViolation` uses `Collectors.toMap(..., (a, b) -> a)`, silently discarding violations on the same property path. Acceptable for typical Bean Validation, but a custom validator with multiple messages per field (e.g., `@Size` + `@Pattern` on the same field) would lose the second message.

**Fix (optional):** Use `(a, b) -> a + "; " + b` to concatenate, or collect into a `Map<String, List<String>>`.

---

**m3. Error response `Content-Type` not explicitly set**

`@ServerExceptionMapper` methods return `Response` without `.type(MediaType.APPLICATION_JSON)`. RESTEasy Reactive + Jackson negotiate this correctly when the request has `Accept: application/json`, but an explicit content type would be defensive against edge cases (e.g., browser requests without JSON Accept header receiving an unexpected content type).

---

### Nit

**N1. `ApiExceptionMapperResource` missing CDI scope annotation**

The class has no `@ApplicationScoped` or `@Singleton`. It works because RESTEasy Reactive discovers `@ServerExceptionMapper` methods at build time regardless. Adding explicit scope would be consistent with the project convention of annotating CDI beans.

---

**N2. `exception` package absent from `layeredArchitecture()` definition**

The exception package is governed by two standalone `ArchTest` rules but not included as a layer in the main `layeredArchitecture()` definition. Adding it would make the architecture map exhaustive and ensure ArchUnit validates both directions in one rule.

---

### Summary

Solid, well-structured implementation that establishes a clean cross-cutting error contract. The exception hierarchy, mapper pattern, and dual-mode frontend extraction all match the design closely. The two highest-risk findings are: (1) a gap in `ErrorSanitizer` that could leak bearer tokens in `Authorization: Bearer <token>` formatted messages, and (2) missing test coverage for the fallback (`INTERNAL_ERROR`) and `ConstraintViolation` mappers — both are spec-mandated scenarios. The `IMPORT_NO_YAML` mapping is dead code due to a design-vs-implementation gap. Recommend fixing M1-M2 before merge and tracking Mo1 as a follow-up.
