## Purpose

Defines the contracts for converting exported 3scale API definitions into Kuadrant, Gateway API, and Istio YAML manifests. Establishes behavior preservation, concurrency safety, REST API stability, and extensibility rules that policy work (#149) must follow.

## Requirements

### Requirement: Conversion output is behavior-preserving

The conversion subsystem MUST produce identical YAML file contents (keys, values, ordering where asserted by tests) for the same 3scale `ApiService` input and conversion options as before this refactor.

#### Scenario: Regression test suite passes

- **WHEN** the full `ConversionServiceTest` suite runs against the refactored code
- **THEN** every YAML string assertion matches the pre-refactor baseline on CI (Linux)

#### Scenario: All output files still generated

- **WHEN** a fully configured 3scale `ApiService` with multiple policies is converted
- **THEN** the result map contains the same file keys (`gateway.yaml`, `httproute.yaml`, `policy.yaml`, `secret.yaml`, `configmap.yaml`, and conditional files such as `destinationrule.yaml`, `serviceentry.yaml`, `telemetry.yaml`, `README.md`) as before the refactor

### Requirement: Conversion is safe under concurrent use

The conversion subsystem MUST support concurrent `convert()` invocations without shared mutable state causing incorrect output or runtime failures.

#### Scenario: Parallel conversion calls succeed

- **WHEN** two threads invoke conversion for different `ApiService` inputs concurrently
- **THEN** both calls complete without exception and each returns YAML matching the sequential baseline for its input

### Requirement: REST conversion API is unchanged

The conversion REST endpoints MUST retain existing request shapes, response shapes, status codes, and error behavior.

#### Scenario: Convert endpoint response shape unchanged

- **WHEN** a client calls the existing convert/preview endpoints with valid credentials and service data
- **THEN** the HTTP status code and response body structure match the pre-refactor contract documented in `docs/api-spec.yml`

### Requirement: Policy lookup is centralized

The conversion subsystem MUST resolve enabled 3scale policies through a single generic lookup mechanism instead of per-policy private finder methods scattered in the orchestrator.

#### Scenario: Enabled policy is found by name

- **WHEN** an `ApiService` has an enabled policy chain entry with a known policy name (e.g. `cors`, `jwt`)
- **THEN** the centralized lookup returns that policy configuration for use by generators and contributors

#### Scenario: Missing policy returns empty

- **WHEN** an `ApiService` does not have the requested policy enabled
- **THEN** the centralized lookup returns absent/empty without error

### Requirement: Resource generation is registry-driven

The conversion subsystem MUST produce each output YAML file through a registered resource generator looked up by registry, not through hardcoded branches in the orchestrator's `convert()` method.

#### Scenario: New output file type registers without orchestrator edit

- **WHEN** a new `ResourceGenerator` implementation is added and registered for a new output file key
- **THEN** conversion includes that file in the result map without adding new conditional branches to the orchestrator class

### Requirement: Multi-policy resources use contributor aggregation

For output files that combine fragments from multiple unrelated 3scale policies (HTTPRoute, AuthPolicy, Secret), the conversion subsystem MUST aggregate contributions through a contributor pattern against a shared builder rather than inline conditional chains in a single generator method.

#### Scenario: HTTPRoute reflects multiple policy sources

- **WHEN** an `ApiService` has both CORS and header-modification policies enabled
- **THEN** the generated `httproute.yaml` contains fragments attributable to each policy, produced by separate contributors

#### Scenario: New policy contributor does not require orchestrator change

- **WHEN** a new contributor is registered for an existing multi-policy resource builder
- **THEN** the parent generator incorporates its fragment without modifying the conversion orchestrator

### Requirement: Conversion options are explicit

The conversion subsystem MUST pass namespace, backend URL, hostname overrides, and related flags through a single explicit options object rather than growing `convert()` overload parameter lists.

#### Scenario: Options carry namespace and overrides

- **WHEN** conversion is invoked with namespace and optional backend URL or hostname overrides
- **THEN** all generators and contributors receive the same options context without additional positional parameters on the orchestrator
