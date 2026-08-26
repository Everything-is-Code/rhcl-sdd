## JaCoCo coverage (`service/generator/`)

| Metric | Baseline (pre-change) | After (#210) |
|--------|----------------------|--------------|
| Lines | ~40% (19/25 files with zero dedicated tests; only `ResourceGeneratorRegistry` + `IsolatedGeneratorTest` partial) | **95.9%** (421/439) |
| Branches | ~25% (estimated) | **65.5%** (127/194) |

Measured with `mvn clean -Dtest='!PlaywrightE2EIT' verify` and merged JaCoCo report (`target/jacoco-merged-report/jacoco.xml`). Conditional generators require `@QuarkusTest` + `@Inject` for JaCoCo to record `generate()` under Quarkus offline instrumentation; plain `new Generator()` covers `applies()` only in the merged report.

## Context

See [proposal.md](proposal.md) for motivation. The `service/generator/` package has 25 source files: 19 concrete generators, 4 factory classes (`Manual*Factory`), 1 interface (`ResourceGenerator`), and 1 registry (`ResourceGeneratorRegistry` — already tested). Each generator is a CDI bean injected with contributors and/or support classes. Generators produce one K8s YAML output file each via `generate(ConversionContext)` and expose `shouldGenerate(ConversionContext)` to control conditional inclusion. Existing indirect tests verify CDI wiring but not YAML output correctness.

## Goals / Non-Goals

**Goals:**
- Raise JaCoCo line+branch coverage for `service/generator/` package to ≥70%
- Create one dedicated test per concrete generator (19 tests total)
- Verify both `shouldGenerate()` conditions and `generate()` YAML structural correctness
- Follow the existing `@QuarkusTest` pattern matching established test infrastructure

**Non-Goals:**
- Refactoring production generator code (only add tests)
- Testing factory classes (trivial CDI producers) or the `ResourceGenerator` interface (no logic)
- Achieving 100% coverage (diminishing returns on edge-case branches)
- Modifying Codecov configuration or thresholds

## Decisions

### 1. Use `@QuarkusTest` (not plain JUnit) for all generator tests

**Rationale:** Generators are CDI beans with injected collaborators (contributors, support classes like `ConversionYamlSupport`, `HttpRouteSupport`, `AuthPolicySupport`). Testing them in isolation with manual construction would require wiring dozens of dependencies and miss CDI-related regressions. Quarkus reuses the application context across test classes, so startup cost is paid once.

**Alternative considered:** Plain JUnit 5 with `new GeneratorX(mockDep1, mockDep2, …)` — rejected because generators have deep dependency trees and CDI qualifiers that are error-prone to replicate manually.

### 2. Mock `ConversionContext` with controlled data, not full service mocks

**Rationale:** `ConversionContext` is a data holder built by `ConversionService`. Tests can construct it directly with a controlled `ApiService`, `Backend`, `ConversionOptions`, policies list, etc. This avoids mocking the entire export/conversion pipeline while giving full control over generator inputs.

**Approach:** Build `ConversionContext` programmatically in each test with:
- A minimal `ApiService` with required fields (name, systemName, backends, policies, metrics)
- A `Backend` with private/public base URLs
- `ConversionOptions` with target namespace, gateway name, DNS hostname (when relevant)
- Policy lists tailored to trigger/skip conditional generators

### 3. YAML assertions verify structural elements, not exact string equality

**Rationale:** Per `AGENTS.md`, CRLF whitespace differences between Windows and Linux cause false assertion failures on exact YAML string matches. Additionally, exact string matching is brittle against formatting changes that don't affect semantics.

**Approach:** Parse generated YAML (or use `assertThat(...).contains(...)` on key fragments) to verify:
- Correct `apiVersion` and `kind`
- Expected `metadata.name` and `metadata.namespace`
- Key `spec` sections present and structurally correct
- Conditional sections present/absent based on input

**Alternative considered:** Snapshot testing (golden files) — rejected because of CRLF issues on Windows and high maintenance cost when formatting changes.

### 4. Priority order: "Always" generators first (7), then conditional (12)

**Rationale:** The 7 "always" generators (`GatewayGenerator`, `HttpRouteGenerator`, `AuthPolicyGenerator`, `SecretGenerator`, `ConfigMapGenerator`, `ApiProductGenerator`, `ReadmeGenerator`) execute for every conversion and have the highest user impact. Testing them first gives the best ROI toward the ≥70% target.

### 5. Each test class verifies both `shouldGenerate()` and `generate()`

**Rationale:** These are the two contract methods of `ResourceGenerator`. For "always" generators, `shouldGenerate()` should always return `true`; for conditional generators, tests must exercise both the `true` and `false` paths to cover the condition logic.

**Structure per test class:**
- `shouldGenerate_returnsTrue_whenConditionMet()` — verifies the trigger condition
- `shouldGenerate_returnsFalse_whenConditionNotMet()` — verifies the skip condition (conditional generators only)
- `generate_producesValidYaml()` — verifies YAML structure with a representative input
- Additional methods for variant inputs if the generator has branching logic

## Risks / Trade-offs

- **[Risk] Contributor ordering affects YAML output** → Generators that aggregate contributors (HttpRoute, AuthPolicy, Secret) produce YAML whose internal structure depends on contributor ordering. Tests should assert the presence of expected sections without asserting their order, or use `ContributorOrdering` to predict order.
- **[Risk] ConversionContext construction complexity** → Some generators read deeply nested fields. Creating test builder helpers or a shared `TestConversionContextFactory` reduces boilerplate across 19 test classes.
- **[Risk] Quarkus test startup time** → 19 new test classes, but Quarkus reuses the CDI context. Marginal CI impact is minimal.
- **[Trade-off] ≥70% target vs. full branch coverage** → Some generators have branches for rare 3scale configurations (e.g., TLS with specific cert formats). Covering the common paths reaches the target; exotic branches are lower priority.
