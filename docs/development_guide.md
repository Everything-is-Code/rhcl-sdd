# Development Guide — Migration Toolkit

## Workspace Layout

Open both repos in Cursor via `rhcl-sdd.code-workspace` in this store (sibling layout).

```bash
cd migration-toolkit-sdd
# Open rhcl-sdd.code-workspace in Cursor
```

Store id: `rhcl-sdd` — `openspec list --store rhcl-sdd`

## Prerequisites

| Tool | Version |
|------|---------|
| Java | 21+ |
| Maven | 3.9+ |
| Node.js | 22 (frontend) |
| npm | 9+ |
| OpenSpec CLI | `@fission-ai/openspec` (global) |
| `oc` | For cluster deploy |

## Local Backend

```bash
cd ../migration-toolkit-rhcl/backend
mvn quarkus:dev
```

PostgreSQL on `localhost:5432` (see `application.properties`).

## Local Frontend

```bash
cd ../migration-toolkit-rhcl/frontend
npm install --legacy-peer-deps
VITE_API_URL=http://localhost:8080 npm run dev
```

## Tests (required before PR handback)

```bash
cd ../migration-toolkit-rhcl/backend && mvn test
cd ../migration-toolkit-rhcl/backend && mvn verify
cd ../migration-toolkit-rhcl/frontend && npm run typecheck
cd ../migration-toolkit-rhcl/frontend && npm test
```

## OpenSpec Store

This folder is registered as store id `migration-toolkit-sdd`.

```bash
openspec store list
openspec list --store migration-toolkit-sdd
openspec new change my-change --store migration-toolkit-sdd
```

## SDD Commands (Cursor)

| Command | Purpose |
|---------|---------|
| `/opsx-explore` | Explore problem space |
| `/opsx-propose` | Create change artifacts |
| `/opsx-apply` | Implement tasks in product repo |
| `/opsx-archive` | Archive completed change |
| `/opsx-sync` | Sync specs |

## Git Conventions

- Branch from `main`: `feature/<issue>-short-description`
- Reference GitHub issues in commits and PRs
- Never amend after PR is open; commit forward
- Tests in same PR as code
- No secrets in commits

## Deploy (reference)

OpenShift S2I + `deploy/install.sh` — see product README. Use `NAMESPACE_PLACEHOLDER` in manifests.
