## MODIFIED Requirements

### Requirement: REST conversion API is unchanged

The conversion REST endpoints MUST retain existing request shapes, response shapes for successful conversions, and status codes. Error responses from conversion endpoints MUST now use the unified error envelope (`{ "error": { "code", "message", "details" } }`) instead of ad-hoc `Map.of("error", ...)` shapes.

Per-service conversion failures within a bulk convert call (partial results inside a 200 response) remain as typed result items — they are NOT transport-level errors and MUST NOT use the error envelope.

#### Scenario: Convert endpoint response shape unchanged

- **WHEN** a client calls the existing convert/preview endpoints with valid credentials and service data
- **THEN** the HTTP status code and response body structure match the pre-refactor contract

#### Scenario: Convert endpoint error uses unified envelope

- **WHEN** a client calls the convert endpoint and a global failure occurs (e.g. 3scale unreachable before the service loop)
- **THEN** the response uses the unified error envelope with an appropriate error code (e.g. `THREESCALE_CLIENT_ERROR`) instead of `{"error": "<message>"}`

#### Scenario: Per-service partial failure does not use envelope

- **WHEN** a bulk conversion call succeeds for some services but fails for one
- **THEN** the overall response is HTTP 200 with per-service status in the results list; the failing service entry contains `status: "FAILED"` and a message, not the error envelope
