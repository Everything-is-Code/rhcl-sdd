## 1. Baseline Measurement

- [x] 1.1 Create feature branch `feature/210-test-coverage-generators` from `main` (or from PR-3 branch if not yet merged) and verify `mvn -Dtest='!PlaywrightE2EIT' verify` passes green
- [x] 1.2 Run JaCoCo report and record current line+branch coverage for the `service/generator/` package; identify which generators have zero dedicated coverage

## 2. "Always" Generators (7 tests — highest priority)

- [x] 2.1 Create `GatewayGeneratorTest.java` — verify `shouldGenerate()` returns `true`, `generate()` produces Gateway YAML with correct apiVersion (`gateway.networking.k8s.io/v1`), kind, metadata, gatewayClassName, listeners section
- [x] 2.2 Create `HttpRouteGeneratorTest.java` — verify `shouldGenerate()` returns `true`, `generate()` produces HTTPRoute YAML with correct parentRefs, hostnames, rules with backendRefs and path matches from mapping rules
- [x] 2.3 Create `AuthPolicyGeneratorTest.java` — verify `shouldGenerate()` returns `true`, `generate()` produces AuthPolicy YAML with correct apiVersion (`kuadrant.io/v1`), kind, targetRef pointing to HTTPRoute, authentication section
- [x] 2.4 Create `SecretGeneratorTest.java` — verify `shouldGenerate()` returns `true`, `generate()` produces Secret YAML with correct metadata, type `Opaque`, data section with expected keys from API key / OIDC credentials
- [x] 2.5 Create `ConfigMapGeneratorTest.java` — verify `shouldGenerate()` returns `true`, `generate()` produces ConfigMap YAML with correct metadata and data entries
- [x] 2.6 Create `ApiProductGeneratorTest.java` — verify `shouldGenerate()` returns `true`, `generate()` produces ApiProduct YAML with correct custom resource structure
- [x] 2.7 Create `ReadmeGeneratorTest.java` — verify `shouldGenerate()` returns `true`, `generate()` produces Markdown README with expected sections (overview, resources, notes)

## 3. Conditional Generators (12 tests)

- [x] 3.1 Create `ApiKeyGeneratorTest.java` — verify `shouldGenerate()` returns `true` only when API key authentication is present, `generate()` produces correct YAML
- [x] 3.2 Create `ServiceEntryGeneratorTest.java` — verify `shouldGenerate()` returns `true` only when external backend URLs are detected, `generate()` produces ServiceEntry YAML
- [x] 3.3 Create `DestinationRuleGeneratorTest.java` — verify `shouldGenerate()` condition and generated DestinationRule YAML structure
- [x] 3.4 Create `TelemetryGeneratorTest.java` — verify `shouldGenerate()` condition and generated Telemetry YAML structure
- [x] 3.5 Create `RateLimitPolicyGeneratorTest.java` — verify `shouldGenerate()` returns `true` only when rate limits exist in plan limits, `generate()` produces RateLimitPolicy YAML with correct limits and counters
- [x] 3.6 Create `LoggingEnvoyFilterGeneratorTest.java` — verify `shouldGenerate()` condition (logging policy present) and generated EnvoyFilter YAML
- [x] 3.7 Create `UrlRewritingEnvoyFilterGeneratorTest.java` — verify `shouldGenerate()` condition (URL rewriting policy present) and generated EnvoyFilter YAML with correct Lua filter configuration
- [x] 3.8 Create `ContentLimitsEnvoyFilterGeneratorTest.java` — verify `shouldGenerate()` condition (content limits policy present) and generated EnvoyFilter YAML
- [x] 3.9 Create `RetryEnvoyFilterGeneratorTest.java` — verify `shouldGenerate()` condition (retry policy present) and generated EnvoyFilter YAML with retry configuration
- [x] 3.10 Create `AuthorizationPolicyGeneratorTest.java` — verify `shouldGenerate()` condition (IP check or referrer policy present) and generated Istio AuthorizationPolicy YAML
- [x] 3.11 Create `TlsPolicyGeneratorTest.java` — verify `shouldGenerate()` returns `true` only when TLS/HTTPS listener is configured, `generate()` produces TLSPolicy YAML with correct issuerRef
- [x] 3.12 Create `DnsPolicyGeneratorTest.java` — verify `shouldGenerate()` returns `true` only when DNS hostname is provided in options, `generate()` produces DNSPolicy YAML with correct routingStrategy

## 4. Verification

- [x] 4.1 Run `mvn -Dtest='!PlaywrightE2EIT' verify` and confirm all tests pass green with zero failures
- [x] 4.2 Open JaCoCo HTML report and verify `service/generator/` package line coverage is ≥70%
- [x] 4.3 Run Checkstyle verification and confirm zero violations in new test files
