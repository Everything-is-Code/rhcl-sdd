## Context

See [proposal.md](proposal.md) for motivation. The `controller` package has 13 source files with 12 having dedicated tests (`DefaultsController` is the gap). Existing tests use `@QuarkusTest` + REST-assured + `@InjectMock` / `@MockitoConfig(convertScopes = true)` for CDI beans. JaCoCo coverage is generated via `mvn -Dtest='!PlaywrightE2EIT' verify` with a merged report at `backend/target/jacoco-merged-report/`.

## Goals / Non-Goals

**Goals:**
- Raise JaCoCo line+branch coverage for `controller` package from ~83% to ≥90%
- Create `DefaultsControllerTest` covering the single untested controller
- Extend existing controller tests to exercise uncovered branches (error paths, null inputs, boundary conditions)
- Keep all tests as `@QuarkusTest` integration tests matching the established pattern

**Non-Goals:**
- Refactoring production controller code (only add tests)
- Achieving 100% coverage (diminishing returns on trivial branches)
- Touching packages outside `controller/` (those are separate PRs per #210 strategy)
- Modifying Codecov configuration or thresholds

## Decisions

### 1. Use `@QuarkusTest` integration tests only (not plain JUnit)

**Rationale:** All 12 existing controller tests are `@QuarkusTest` + REST-assured. Mixing unit tests would break consistency and miss CDI/JAX-RS wiring issues. The startup cost is paid once per test class since Quarkus reuses the app context.

**Alternative considered:** Plain JUnit 5 with mocked `Response` — rejected because controller tests should verify the full HTTP layer (status codes, content types, error envelope serialization).

### 2. `DefaultsControllerTest` uses Quarkus `@ConfigProperty` test overrides

**Rationale:** `DefaultsController` reads `threescale.default.url` and `threescale.default.token` from `Optional<ConfigProperty>`. Tests need to verify behavior with: (a) both properties set, (b) both absent/empty, (c) blank strings that get filtered by `.filter(s -> !s.isBlank())`. Quarkus test profiles or `@TestProfile` with config overrides are the standard way.

**Approach:** Use two test profiles:
- Default profile: properties unset → `configured: false`, url/token: `null`
- Custom `@TestProfile`: properties set → `configured: true`, url/token: non-null

**Alternative considered:** `@InjectMock` on the config fields — rejected because `@ConfigProperty Optional<String>` is not a CDI bean and cannot be easily mocked. Profile-based override is cleaner.

### 3. Focus on error-path and edge-case branches in existing tests

**Rationale:** The existing 12 test files cover happy paths well. The coverage gap is mostly in:
- Error handling paths (`catch` blocks, error envelope responses)
- Null/empty input validation
- Partial result scenarios (bulk operations returning mixed success/failure)
- Conditional branches in controllers that delegate to services

Running JaCoCo locally first and identifying uncovered lines guides the exact test methods to add.

### 4. Run JaCoCo first, then write tests (measure-then-act)

**Rationale:** Instead of guessing which branches to cover, generate the JaCoCo HTML report, inspect per-controller coverage, and target the red/yellow lines. This avoids writing redundant tests.

## Risks / Trade-offs

- **[Risk] Quarkus test startup time** → Each new test class adds ~0s marginal (Quarkus reuses context). Only 1 new class (`DefaultsControllerTest`), rest are methods in existing classes. Minimal CI impact.
- **[Risk] `DefaultsController` config override requires `@TestProfile`** → If creating a separate profile is cumbersome, use `application.properties` test config with `%test.threescale.default.url=...` for the "configured" case, and a dedicated `@TestProfile` that unsets them for the "unconfigured" case. Alternatively, use `@QuarkusTestResource` or `quarkus.test.profile` system property.
- **[Risk] CRLF whitespace issues on Windows** → Per `AGENTS.md`, string assertions on YAML output are whitespace-sensitive. Controller tests assert JSON, not YAML, so this risk is low. Trust CI (Linux) for final validation.
- **[Trade-off] Not reaching 100%** → Some branches (e.g., `toString()`, serialization edge cases) have diminishing returns. The 90% target balances ROI with effort.
