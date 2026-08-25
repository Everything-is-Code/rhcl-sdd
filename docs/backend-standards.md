---
description: Backend standards — Quarkus 3.27, Java 21, ArchUnit layering, Fabric8, Flyway V1–V9.
globs: ["../migration-toolkit-rhcl/backend/**"]
alwaysApply: true
---

# Backend Standards — Migration Toolkit

> Ground truth audit: `../migration-toolkit-rhcl/TECHNICAL_SPECIFICATIONS.md` §1–5.  
> Layering/tests detail: `../migration-toolkit-rhcl/.cursor/rules/testing-standards.mdc`  
> Data model: `../migration-toolkit-rhcl/.cursor/rules/data-model.mdc`

## Stack

| Component | Version / choice |
|-----------|------------------|
| Runtime | Java 21 |
| Framework | Quarkus **3.27.5.1** |
| REST | RESTEasy Reactive + Jackson |
| Persistence | Hibernate ORM Panache + PostgreSQL |
| Migrations | Flyway **V1–V9** |
| K8s | Fabric8 6.7.x (`quarkus-kubernetes-client`) |
| Validation | Hibernate Validator (`@Valid` on DTOs) |
| YAML | SnakeYAML 2.6 |
| OpenAPI | SmallRye OpenAPI + `/q/swagger-ui` |
| Testing | JUnit 5, REST-assured, Mockito, H2 test profile, **ArchUnit**, Playwright E2E, Awaitility |
| Static analysis | Checkstyle, PMD, JaCoCo (unit + Quarkus merged) |
| Build | Maven 3.9+ |

## Package layout (44 source files)

```
com.redhat.migrationtoolkit.rhcl/
  controller/   14 JAX-RS endpoints
  service/      6 @ApplicationScoped services
  entity/       3 Panache entities
  dto/          7 request/response shapes (no Dto suffix)
  model/        11 internal 3scale graph models
  client/       1 ThreeScale REST client interface
  util/         Messages, ConversionConstants
```

## ArchUnit layering (do not violate)

Enforced in `backend/src/test/architect/.../ArchitectureTest.java`:

- `controller → service → client`; controllers **must not** depend on `client` directly
- `*Controller` in `controller..` with `@Path`
- `*Service` in `service..`/`model..`, `@ApplicationScoped`, no `@Path`
- `*Entity` in `entity..`, `@Entity`, no dependency on `controller..`
- `client..` types are interfaces
- No `System.out` / `java.util.logging`

## DI and API patterns (current code)

- **Field injection** with `@Inject` everywhere — match existing style, don't introduce constructor injection ad hoc
- `@Transactional` on **controller** methods that persist entities (not in services today)
- Responses: typed DTOs or records nested in controllers — no shared envelope yet
- Errors: per-endpoint `try/catch` returning `Map.of("error", ...)` — **three inconsistent shapes** today (see TECHNICAL_SPECIFICATIONS §5.8)
- File upload: `@RestForm("file") FileUpload` in `ImportController`
- Tokens: `Authorization: Bearer <token>` only — never query string

## Key services

| Service | Notes |
|---------|-------|
| `ConversionService` | ~1300 lines — god class; target #40 Strategy/Registry/Contributor |
| `ThreeScaleExportService` | TTL caches + token fingerprint (SHA-256) on cache keys |
| `ClusterVersionService` | Request coalescing via `CompletableFuture` map |
| `CompatibilityService` | Scoring JWT/rewrite/Lua/SOAP |
| `ValidationService` | YAML/CRD validation; flags `REPLACE_ME` placeholder |
| `ApplyController` flow | SSA + scoped RBAC Role/RoleBinding |

## Testing

```bash
cd ../migration-toolkit-rhcl/backend && mvn test      # unit + ArchUnit
cd ../migration-toolkit-rhcl/backend && mvn verify    # + IT, JaCoCo, Checkstyle, PMD
```

- Test method naming: `methodUnderTest_condition_expectedResult`
- Mock at HTTP/K8s boundaries; `@QuarkusTest` + REST-assured for controllers
- `ConversionServiceTest`: YAML string assertions — **CRLF false failures on Windows** (`core.autocrlf=true`); trust Linux CI

## Security

- Never commit tokens/kubeconfigs
- `ConversionConstants.CREDENTIAL_PLACEHOLDER = "REPLACE_ME"` in generated secrets
- `ClusterVersionService.sanitize()` redacts tokens from logs/errors
- Backend has **no auth layer** — trusts caller-supplied 3scale/cluster credentials (admin tool assumption)

## Logging

`org.jboss.logging.Logger` named `LOG`; use `infof`/`warnf`/`debugf` — never string concat.
