---
description: Backend standards for Migration Toolkit — Quarkus 3.27, Java 21, REST, Fabric8, Flyway.
globs: ["../migration-toolkit-rhcl/backend/**"]
alwaysApply: true
---

# Backend Standards — Migration Toolkit

## Stack

| Component | Version / choice |
|-----------|------------------|
| Runtime | Java 21 (OpenJDK) |
| Framework | Quarkus 3.27.5.1 |
| REST | RESTEasy Reactive |
| Persistence | Hibernate ORM Panache + PostgreSQL |
| Migrations | Flyway (`V1`–`V3`) |
| K8s client | Fabric8 6.7.x |
| OpenAPI | SmallRye OpenAPI + Swagger UI (`/q/swagger-ui`) |
| Build | Maven 3.9+ |

## Package Layout

```
backend/src/main/java/com/redhat/migrationtoolkit/rhcl/
  controller/   REST resources
  service/      Business logic (ConversionService, ThreeScaleExportService, …)
  client/       3scale HTTP client
  dto/          Request/response DTOs
  model/        Domain models (ApiService, Policy, MappingRule, …)
  entity/       JPA entities (ProjectEntity, ConversionHistoryEntity, …)
  util/         Messages, ConversionConstants
```

## Layering Rules

- **Controllers**: thin — validate input, resolve locale via `Messages`, delegate to services.
- **Services**: orchestration, 3scale/K8s I/O, conversion logic.
- **Entities / models**: persistence and domain shapes; no HTTP concerns.
- **DTOs**: API boundary only.

## API Conventions

- Base path: `/api/*`
- JSON request/response
- User-facing strings via `Messages` + `Accept-Language` (EN default, JA supported)
- OpenAPI maintained by Quarkus annotations; keep in sync with `docs/api-spec.yml` in this store

## 3scale Integration

`ThreeScaleClient` / `ThreeScaleExportService` call Admin API:

- `GET /admin/api/services.json`
- Backends, proxy configs, policies, mapping rules, metrics, auth

Respect pagination and avoid N+1 patterns (see issue #169).

## Cluster Operations

- Apply via Server-Side Apply (`ApplyController`)
- Auto RBAC provisioning in target namespace
- Gateway info from live cluster (`GatewayInfoController`)

## Testing

```bash
cd ../migration-toolkit-rhcl/backend && mvn test
cd ../migration-toolkit-rhcl/backend && mvn verify
```

- JUnit 5 + Quarkus test extensions
- Mock at HTTP/K8s boundaries, not internal service logic under test
- Conversion tests: assert YAML structure; be aware of CRLF on Windows

## Security

- Never commit tokens, kubeconfigs, or `.env` secrets
- Redact secrets in logs and exported YAML where policies require it

## Error Handling

- Use domain-appropriate HTTP status codes
- Localized error messages through `Messages` bean
- Log structured context (service id, namespace) without secrets
