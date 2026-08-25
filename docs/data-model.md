# Data Model — Migration Toolkit

> Field-level spec: `../migration-toolkit-rhcl/.cursor/rules/data-model.mdc`  
> Audit: `../migration-toolkit-rhcl/TECHNICAL_SPECIFICATIONS.md` §4

PostgreSQL via Flyway in `backend/src/main/resources/db/migration/` (**V1–V9**).

## Entities (Panache/JPA)

### ProjectEntity

| Field | Notes |
|-------|-------|
| id | PK |
| name | |
| threescaleUrl | Admin portal URL |
| tenant | |
| createdAt, updatedAt | |

### ConversionHistoryEntity

| Field | Notes |
|-------|-------|
| id | PK |
| project_id | FK → Project |
| source | `CONVERT` \| `IMPORT` (string today — consider enum #8 in TECH_SPEC) |
| serviceId, serviceName | CONVERT only |
| namespace | Target OpenShift namespace |
| status | `COMPLETED` \| `PARTIAL` \| `FAILED` \| `IN_PROGRESS` |
| compatibilityScore | CONVERT only |
| totalCount, successCount, failureCount | Apply stats |
| failureDetails | JSON text — `[{fileName, kind, name, error}]` |
| exportedYaml | JSON text — `{filename → yaml}` post-apply |
| yamlContent | Full generated YAML (CONVERT) |
| packageName | |
| createdAt | |

### AppSettingsEntity

Generic key/value store: `settings_key` (PK), `value` (TEXT), `updatedAt`.

## Relationships

```
Project 1---* ConversionHistory
```

## Domain models (non-JPA, `model/`)

3scale graph: `ApiService`, `Backend`, `MappingRule`, `Metric`, `Policy`, `Authentication`, `Application`, `ApplicationPlan`, `Route`, `Project`, `CompatibilityItem`, `CompatibilityResult`.

## Flyway

Versions **V1–V9** — do not document only V1–V3; check `db/migration/` for current truth.

## Suggested improvements (not implemented)

- `@Enumerated(STRING)` for status/source
- Native `jsonb` for JSON columns (see TECHNICAL_SPECIFICATIONS §7 #8)

## Policy mapping

`docs/conversion-architecture.md` + GitHub #149.
