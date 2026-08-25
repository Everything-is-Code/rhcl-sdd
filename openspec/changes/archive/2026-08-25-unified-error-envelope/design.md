## Context

See proposal.md — Why. The backend currently has zero centralized exception handling. Each controller invents its own error body shape, forcing the frontend to guess the structure. After PR #190 (conversion Strategy+Registry) and PR #191 (frontend component split), the codebase is stable for a cross-cutting error contract.

Current state:
- Backend: every catch block builds `Response.status(...).entity(Map.of("error", ...))` with varying shapes.
- Frontend: `apiErrorMessage()` in `utils/apiError.ts` already handles the legacy fallback chain; all pages use `catch (e: unknown)`.
- No `ExceptionMapper` or `@ServerExceptionMapper` exists anywhere in the codebase.

## Goals / Non-Goals

**Goals:**
- Single predictable error envelope for all HTTP error responses
- Typed exception hierarchy for common backend error categories
- Phase-1 migration of highest-traffic controllers (`ConversionController`, `ApplyController`, `ImportController`)
- Frontend backward-compatible extraction that handles both envelope and legacy shapes
- Fallback mapper that catches all uncaught exceptions from any endpoint
- Frontend i18n mapping: error `code` → localized user-facing message via `react-i18next`, with backend `message` as fallback

**Non-Goals:**
- Migrating all controllers in one change (`ConnectionController`, `SetupController`, `HistoryController`, `ClusterController`, `GatewayInfoController` deferred to phase 2)
- Changing success response shapes (only error responses)
- Changing `ApplyResult` partial-success pattern (stays 200 + list)
- `ConnectionController`'s `{"success": true/false}` success shape (requires separate design)

## Decisions

### D1 — `@ServerExceptionMapper` over `@Provider ExceptionMapper`

**Choice:** RESTEasy Reactive `@ServerExceptionMapper` methods in a single CDI bean.

**Why:** The project uses Quarkus 3.27 with RESTEasy Reactive. `@ServerExceptionMapper` is the idiomatic approach — avoids `@Provider` scanning, integrates with the reactive pipeline, and keeps all mappers co-located in one class.

**Alternative considered:** JAX-RS `@Provider ExceptionMapper<T>` per exception type — more boilerplate, multiple classes, non-reactive.

### D2 — `ApiException` abstract base class (not sealed hierarchy)

**Choice:** An abstract `ApiException extends RuntimeException` base with `code`, `status`, and `details` fields. Concrete subclasses for phase-1 error categories.

```
ApiException (abstract, RuntimeException)
├── code: String
├── status: int
├── details: Map<String, Object>
│
├── ValidationException        (400, VALIDATION_FAILED)
├── ThreeScaleClientException  (502, THREESCALE_CLIENT_ERROR)
├── ClusterApplyException      (500, APPLY_FAILED)
└── ImportParseException       (400, IMPORT_PARSE_ERROR)
```

**Why:** `RuntimeException` because Quarkus reactive pipeline does not propagate checked exceptions well. Abstract (not sealed) to allow phase-2 additions without modifying the base. Minimal hierarchy — only the categories that phase-1 controllers actually throw.

**Alternative considered:** Sealed hierarchy with `permits` — too rigid for incremental migration.

### D3 — Exceptions carry resolved messages; frontend maps codes to i18n

**Choice:** Exceptions carry the already-localized `message` string. The mapper copies `getMessage()` into the envelope. The frontend maintains an `errorCodeMessages` map from `SCREAMING_SNAKE` codes to i18n keys (e.g. `VALIDATION_FAILED` → `t('error.validationFailed')`). When the envelope `code` is recognized, the frontend uses the i18n translation; otherwise it falls back to the backend `message`.

**Why:** The backend already localizes some messages via `messages.get(key, locale)`. Having the frontend also map codes to i18n keys gives a consistent, design-system-aligned UX for known error categories, while the backend `message` serves as a reliable fallback for unexpected or new codes. The i18n key set is small (one per error code) and lives in `en.json` / `ja.json`.

### D4 — Fallback mapper for `Exception.class` is intentional

**Choice:** Register a fallback `@ServerExceptionMapper(Exception.class)` that produces `INTERNAL_ERROR` (500) for any uncaught exception.

**Why:** This changes behavior for all endpoints from day 1 — including non-migrated controllers where unexpected exceptions currently return Quarkus's default HTML/JSON. The change is intentional: even non-migrated endpoints return the predictable envelope for unexpected failures. Existing controller tests must be verified (status codes unchanged; body shape changes).

### D5 — Sanitization reuses existing patterns

**Choice:** The mapper calls a `sanitize(message)` utility (same pattern as `ClusterVersionService.sanitize()`) before writing the `message` field.

**Why:** Prevents token leakage without inventing a new redaction mechanism. The existing `sanitize()` replaces known credential patterns with `[REDACTED]`.

### D6 — Frontend dual-mode extraction

**Choice:** Extend `apiErrorMessage()` to check `typeof error === 'object'` → read `error.message`; otherwise fall through to existing string/legacy handling.

**Why:** Non-migrated endpoints still return legacy shapes. The extraction must handle both until all controllers are migrated. No breaking change for existing callers.

### D7 — Error code → i18n key mapping in frontend

**Choice:** Add an `ERROR_CODE_I18N` map (e.g. `Record<string, string>`) in `utils/apiError.ts` that maps known envelope codes to i18n keys. `apiErrorMessage()` checks: if envelope detected and `code` is in the map → `t(ERROR_CODE_I18N[code])`; else → fall back to `error.message` from the envelope or legacy extraction.

**Why:** Gives the frontend full control over user-facing wording, consistent tone, and bilingual support (EN/JA). The backend `message` is still the safety net for codes the frontend hasn't mapped yet (e.g. new codes added before the frontend catches up). The map is small — one entry per phase-1 error code (`VALIDATION_FAILED`, `THREESCALE_CLIENT_ERROR`, `APPLY_FAILED`, `IMPORT_PARSE_ERROR`, `IMPORT_NO_YAML`, `INTERNAL_ERROR`).

**Alternative considered:** Only use the backend `message` — simpler but loses control over tone and JA translations. Rejected because the SPA already has i18n infrastructure.

## Risks / Trade-offs

**[Risk] Fallback mapper changes error shape for non-migrated endpoints** → Mitigation: verify all existing controller tests pass with the new envelope shape. Status codes don't change; only the body structure changes from Quarkus default to the envelope. This is strictly an improvement.

**[Risk] `ConstraintViolationException` mapper conflicts with Quarkus built-in handling** → Mitigation: Quarkus RESTEasy Reactive allows overriding the default handler. The explicit `@ServerExceptionMapper(ConstraintViolationException.class)` takes precedence. Test with `@Valid` annotated endpoints.

**[Risk] Message sanitization misses new credential patterns** → Mitigation: reuse the proven `sanitize()` approach; add patterns as discovered. Log the full exception with credentials server-side (existing `LOG.warnf` behavior preserved).

**[Trade-off] Phase-1 leaves some controllers un-migrated** → Acceptable: the fallback mapper covers them for unexpected errors; their explicit catch blocks still work as before for expected errors. Phase-2 migrates the rest.

## Migration Plan

1. Add `exception/` package with `ApiException` hierarchy + mapper — no controller changes yet
2. Add fallback mapper — all endpoints now return envelope for uncaught exceptions
3. Migrate phase-1 controllers one at a time: `ImportController` → `ApplyController` → `ConversionController`
4. Update frontend `apiErrorMessage()` with dual-mode extraction + error code → i18n map
5. Add i18n keys for all phase-1 error codes in `en.json` and `ja.json`
6. Verify: `mvn test`, `npm run typecheck`, `npm test`
7. Rollback: revert the mapper bean — controllers fall back to their existing catch blocks
