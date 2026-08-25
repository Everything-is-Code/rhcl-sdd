# Data Model — Migration Toolkit

PostgreSQL schema managed by Flyway in `migration-toolkit-rhcl/backend/src/main/resources/db/migration/`.

## Project

| Column | Type | Notes |
|--------|------|-------|
| id | PK | |
| name | string | |
| namespace | string | Target OpenShift namespace |
| threescaleUrl | string | Admin portal URL |
| createdAt | timestamp | |

## ConversionHistory

| Column | Type | Notes |
|--------|------|-------|
| id | PK | |
| source | enum | `CONVERT` \| `IMPORT` |
| namespace | string | Target namespace |
| serviceId | long? | CONVERT only |
| serviceName | string? | CONVERT only |
| status | enum | `COMPLETED` \| `PARTIAL` \| `FAILED` |
| compatibilityScore | number? | CONVERT only |
| totalCount | int | Resources attempted |
| successCount | int | Applied successfully |
| failureCount | int | Failed resources |
| failureDetails | JSON | `[{fileName, kind, name, error}]` |
| exportedYaml | JSON | `{filename → yaml}` from cluster after apply |
| yamlContent | text? | Full generated YAML (CONVERT) |
| createdAt | timestamp | |

## Flyway Migrations

| Version | Purpose |
|---------|---------|
| V1__init.sql | Project, ConversionHistory |
| V2__add_sequences.sql | Sequence additions |
| V3__import_history.sql | Import history fields |

## Domain Models (non-JPA)

Used for 3scale API shapes and conversion pipeline:

- `ApiService`, `Backend`, `MappingRule`, `Metric`, `Policy`, `Authentication`
- `Application`, `ApplicationPlan`, `Route`, `Project`
- `CompatibilityItem`, `CompatibilityResult`

## 3scale → K8s Mapping

See `docs/conversion-architecture.md` and GitHub epic #149 for policy-level mappings.

Reference architecture docs (legacy program): `rhcl-ai/docs/architecture/3scale-to-cl-mapping.md` — verify against current toolkit behavior.
