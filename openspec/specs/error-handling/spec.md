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

Partial-success responses (HTTP 200/207 with per-item results, e.g. `ApplyResult[]`, `SetupResponse`, per-service conversion results, or gateway `ready: false` on 200) are NOT error responses and MUST NOT use this envelope.

This requirement applies to ALL controllers with error paths, not only a subset. Controllers that only return success (e.g. `DefaultsController`, `ValidationController`) are exempt.

#### Scenario: Validation error returns envelope with field details

- **WHEN** a client sends a request that fails `@Valid` / `ConstraintViolation` checks
- **THEN** the response is HTTP 400 with `code: "VALIDATION_FAILED"` and `details` containing the violated field names and messages

#### Scenario: Upstream 3scale failure returns envelope

- **WHEN** a controller operation fails due to a 3scale API timeout or connection error
- **THEN** the response is HTTP 502 with `code: "THREESCALE_CLIENT_ERROR"` and a sanitized message (no tokens or credentials)

#### Scenario: Uncaught exception returns fallback envelope

- **WHEN** an unexpected exception propagates from any endpoint (migrated or not)
- **THEN** the response is HTTP 500 with `code: "INTERNAL_ERROR"` and a generic message (no stack trace or internal details exposed)

#### Scenario: Connection test failure returns envelope

- **WHEN** a client sends a connection test request and the 3scale server is unreachable or rejects the credentials
- **THEN** the response is HTTP 502 with `code: "CONNECTION_TEST_FAILED"` and the envelope shape; the success path remains `200 { success: true, message }`

#### Scenario: Resource not found returns envelope with resource-specific code

- **WHEN** a client requests a specific resource (history entry, gateway, settings key, cluster route) that does not exist
- **THEN** the response is HTTP 404 with an envelope containing a resource-specific code (e.g. `HISTORY_NOT_FOUND`, `GATEWAY_NOT_FOUND`, `SETTINGS_NOT_FOUND`, `CLUSTER_ROUTE_NOT_FOUND`) and a descriptive message

#### Scenario: Missing required parameters returns VALIDATION_FAILED envelope

- **WHEN** a controller endpoint receives a request missing required query parameters or body fields (e.g. ExportController missing url/token, PackageController missing yamlFiles, GatewayInfoController missing namespace/name)
- **THEN** the response is HTTP 400 with `code: "VALIDATION_FAILED"` and the envelope shape, not a plain-string body

#### Scenario: Cluster domain extraction failure returns envelope

- **WHEN** a client requests the cluster domain and the backend route is found but the domain cannot be extracted from the host
- **THEN** the response is HTTP 404 with `code: "CLUSTER_DOMAIN_EXTRACT_FAILED"` and the envelope shape

#### Scenario: Cluster route host not yet assigned returns envelope

- **WHEN** a client requests the cluster domain and the backend route exists but has no host assigned yet
- **THEN** the response is HTTP 404 with `code: "CLUSTER_ROUTE_HOST_PENDING"` and the envelope shape

#### Scenario: History download failure returns envelope

- **WHEN** a client requests to download a history entry and ZIP creation fails due to an internal error
- **THEN** the response is HTTP 500 with `code: "HISTORY_DOWNLOAD_FAILED"` and the envelope shape with a sanitized message

#### Scenario: SetupController partial success is not an error envelope

- **WHEN** a setup namespace operation partially succeeds (some steps succeed, some fail)
- **THEN** the response is HTTP 207 with a `SetupResponse` body containing per-step `StepResult` entries; the error envelope is NOT used for per-step failures

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

The exception hierarchy MUST include a `NotFoundException` subclass that produces HTTP 404 responses with a caller-specified error code and message.

#### Scenario: Custom exception maps to correct status and code

- **WHEN** a controller throws a `ValidationException` with code `"VALIDATION_FAILED"` and status 400
- **THEN** the mapper produces an HTTP 400 response with `error.code` equal to `"VALIDATION_FAILED"`

#### Scenario: NotFoundException maps to 404 with resource-specific code

- **WHEN** a controller throws a `NotFoundException` with code `"HISTORY_NOT_FOUND"` and status 404
- **THEN** the mapper produces an HTTP 404 response with `error.code` equal to `"HISTORY_NOT_FOUND"`

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

The mapping MUST include entries for all error codes produced by the backend:
`VALIDATION_FAILED`, `THREESCALE_CLIENT_ERROR`, `APPLY_FAILED`, `IMPORT_PARSE_ERROR`, `IMPORT_NO_YAML`, `INTERNAL_ERROR`, `CONNECTION_TEST_FAILED`, `HISTORY_NOT_FOUND`, `HISTORY_DOWNLOAD_FAILED`, `CLUSTER_ROUTE_NOT_FOUND`, `CLUSTER_ROUTE_HOST_PENDING`, `CLUSTER_DOMAIN_EXTRACT_FAILED`, `GATEWAY_NOT_FOUND`, `SETTINGS_NOT_FOUND`.

#### Scenario: Known error code renders localized message

- **WHEN** the frontend receives an envelope with `code: "VALIDATION_FAILED"` and the user's locale is `ja`
- **THEN** the UI displays the Japanese translation from the `ja.json` locale file, not the English backend `message`

#### Scenario: Unknown error code falls back to backend message

- **WHEN** the frontend receives an envelope with a `code` not present in the i18n map
- **THEN** the UI displays the backend `message` string from the envelope as-is

#### Scenario: Legacy error shape still works

- **WHEN** the frontend receives a legacy error response (no envelope, just `{"error": "string"}`)
- **THEN** the UI displays the legacy string — i18n mapping is not attempted

### Requirement: All frontend error catch blocks use i18n-aware extraction

Every production catch block in page components and hooks that handles API errors MUST use `apiErrorI18nMessage()` (or equivalent i18n-aware extraction). Direct calls to the raw `apiErrorMessage()` MUST NOT appear in production catch blocks; `apiErrorMessage()` serves only as the internal fallback within the i18n-aware extraction function.

#### Scenario: Page catch block uses i18n extraction

- **WHEN** a page component catches an API error from any endpoint
- **THEN** the displayed error message is produced by the i18n-aware extraction function, which returns a localized string for recognized codes and the raw message for unrecognized codes

#### Scenario: Gateway polling hook distinguishes terminal errors from transient

- **WHEN** the gateway polling hook receives a 400 or 404 error
- **THEN** polling stops and the i18n-aware error message is displayed
- **WHEN** the gateway polling hook receives a 500 or network error
- **THEN** polling retries according to its existing backoff logic
