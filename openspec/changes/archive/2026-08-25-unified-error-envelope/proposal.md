## Why

The backend has zero `ExceptionMapper` or `@ServerExceptionMapper` classes. Every controller wraps its body in its own `try/catch` and invents a different error shape (`{"error": "..."}`, `{"success": false, "message": "..."}`, typed result records). The frontend must probe multiple possible shapes per API call via `err.response?.data?.error || err.response?.data?.message || err.message`. This fragile guessing makes error UX inconsistent and new endpoint integration error-prone.

With PR #190 (conversion Strategy+Registry) and PR #191 (frontend component split) now merged, both backend controllers and frontend error handling are stable enough to introduce a cross-cutting error contract without merge conflicts.

## What Changes

- Introduce a small `ApiException` hierarchy (`ValidationException`, `ThreeScaleClientException`, `ClusterApplyException`, `ImportParseException`) in a new `exception/` package, each carrying an HTTP status and a stable error `code` string.
- Add `@ServerExceptionMapper` handlers that translate those exceptions — plus `ConstraintViolationException` and a fallback `Exception` — into a single error envelope: `{ "error": { "code", "message", "details" } }`.
- Migrate phase-1 controllers (`ConversionController`, `ApplyController`, `ImportController`) to throw typed exceptions instead of manually building `Response.status(...).entity(Map.of(...))`.
- Update frontend `apiErrorMessage()` to read the new nested envelope shape while preserving backward compatibility with legacy shapes from non-migrated endpoints.
- Update `ArchitectureTest` to cover the new `exception` package layering.

## Capabilities

### New Capabilities
- `error-handling`: Defines the unified error response envelope contract, exception hierarchy, mapper behavior, and frontend extraction rules.

### Modified Capabilities
- `conversion-pipeline`: The conversion REST API error responses change from ad-hoc shapes to the unified envelope. Adds a requirement that conversion errors are expressed through typed exceptions and the standard error envelope.

## Impact

- **Backend controllers** (phase 1): `ConversionController`, `ApplyController`, `ImportController` — catch blocks replaced with typed throws.
- **Backend new package**: `com.redhat.migrationtoolkit.rhcl.exception/` — exception classes + mapper.
- **Backend tests**: New `ApiExceptionMapperTest`; existing controller tests updated to assert envelope shape.
- **Frontend**: `utils/apiError.ts` extended; `apiError.test.ts` new envelope case.
- **ArchUnit**: `ArchitectureTest.java` updated for `exception` layer.
- **SDD docs**: `technical-specifications.md` §5.8 updated post-merge.
- **No breaking changes** for non-migrated endpoints: the fallback mapper produces the same envelope for uncaught exceptions, which is an improvement over the current Quarkus default HTML/JSON.
