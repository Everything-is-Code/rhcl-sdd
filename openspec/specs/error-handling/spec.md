## Purpose

Defines the unified error response contract for the Migration Toolkit REST API, ensuring every HTTP error response follows a single predictable envelope that frontend and external consumers can parse with one code path.

## Requirements

### Requirement: All HTTP error responses use a unified envelope

Every REST endpoint MUST return HTTP error responses (4xx, 5xx) in the following JSON shape:

```json
{ "error": { "code": "<SCREAMING_SNAKE>", "message": "<human-readable>", "details": {} } }
```

- `code`: a stable `SCREAMING_SNAKE_CASE` string identifying the error category.
- `message`: a human-readable, token-safe string suitable for direct UI display.
- `details`: an optional object carrying structured context (field names, constraint violations, resource identifiers).

Partial-success responses (HTTP 200 with per-item results, e.g. `ApplyResult[]`) are NOT error responses and MUST NOT use this envelope.

#### Scenario: Validation error returns envelope with field details

- **WHEN** a client sends a request that fails `@Valid` / `ConstraintViolation` checks
- **THEN** the response is HTTP 400 with `code: "VALIDATION_FAILED"` and `details` containing the violated field names and messages

#### Scenario: Upstream 3scale failure returns envelope

- **WHEN** a controller operation fails due to a 3scale API timeout or connection error
- **THEN** the response is HTTP 502 with `code: "THREESCALE_CLIENT_ERROR"` and a sanitized message (no tokens or credentials)

#### Scenario: Uncaught exception returns fallback envelope

- **WHEN** an unexpected exception propagates from any endpoint (migrated or not)
- **THEN** the response is HTTP 500 with `code: "INTERNAL_ERROR"` and a generic message (no stack trace or internal details exposed)

### Requirement: Error messages never expose secrets

The `message` field in the error envelope MUST NOT contain bearer tokens, access tokens, API keys, or other credential material, even if the originating exception message includes them.

#### Scenario: Token in upstream error is sanitized

- **WHEN** a 3scale client exception contains an access token in its message
- **THEN** the envelope `message` replaces or redacts the token before returning to the client

### Requirement: Backend exceptions carry error metadata

Each typed exception MUST carry:
- An HTTP status code (used by the mapper for the response status)
- A stable error `code` string (used in the envelope `code` field)
- An optional `details` map

#### Scenario: Custom exception maps to correct status and code

- **WHEN** a controller throws a `ValidationException` with code `"VALIDATION_FAILED"` and status 400
- **THEN** the mapper produces an HTTP 400 response with `error.code` equal to `"VALIDATION_FAILED"`

### Requirement: Frontend error extraction handles both envelope and legacy shapes

The frontend error extraction utility MUST read the unified envelope format (`error.code` + `error.message` when `error` is an object) AND continue to handle legacy shapes (`error` as string, `message` as string, `success: false` patterns) from non-migrated endpoints.

#### Scenario: New envelope shape extracted correctly

- **WHEN** an axios error has `response.data.error` as an object with `{ code, message }`
- **THEN** the extraction utility returns the `message` string

#### Scenario: Legacy string error still extracted

- **WHEN** an axios error has `response.data.error` as a plain string
- **THEN** the extraction utility returns that string (backward compatible)

#### Scenario: Legacy success-false shape still extracted

- **WHEN** an axios error has `response.data` as `{ success: false, message: "..." }`
- **THEN** the extraction utility returns the `message` string

### Requirement: ConstraintViolationException produces the unified envelope

The system MUST intercept `ConstraintViolationException` (from `@Valid` annotations) and produce the unified envelope instead of the Quarkus/RESTEasy default JSON shape.

#### Scenario: Bean validation failure returns VALIDATION_FAILED envelope

- **WHEN** a `@Valid`-annotated request body has constraint violations
- **THEN** the response is HTTP 400 with `code: "VALIDATION_FAILED"` and `details` listing each violated property and its constraint message

### Requirement: Frontend maps error codes to localized messages

The frontend MUST maintain a mapping from known envelope error codes to i18n translation keys. When the extraction utility detects an envelope with a recognized `code`, it MUST return the localized i18n string instead of the raw backend `message`. For unrecognized codes, it MUST fall back to the backend `message` field.

The i18n keys MUST exist in both `en.json` and `ja.json` locale files.

#### Scenario: Known error code renders localized message

- **WHEN** the frontend receives an envelope with `code: "VALIDATION_FAILED"` and the user's locale is `ja`
- **THEN** the UI displays the Japanese translation from the `ja.json` locale file, not the English backend `message`

#### Scenario: Unknown error code falls back to backend message

- **WHEN** the frontend receives an envelope with a `code` not present in the i18n map
- **THEN** the UI displays the backend `message` string from the envelope as-is

#### Scenario: Legacy error shape still works

- **WHEN** the frontend receives a legacy error response (no envelope, just `{"error": "string"}`)
- **THEN** the UI displays the legacy string — i18n mapping is not attempted
