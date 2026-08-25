---
description: Core development rules for Migration Toolkit RHCL — single source of truth for all AI agents in this store.
alwaysApply: true
---

## 1. Core Principles

- **Small tasks, one at a time**: Baby steps; never skip ahead of the current OpenSpec task.
- **Test-first for new behavior**: Failing tests before implementation (JUnit 5 + ArchUnit backend, Vitest frontend).
- **Type safety**: Java explicit types; TypeScript `strict: true`.
- **Clear naming**: 3scale / Kuadrant / Gateway API domain language.
- **Incremental changes**: Focused PRs; no unrelated hygiene in feature work.
- **Spec is source of truth**: Code must trace to OpenSpec artifacts in this store after `/opsx-propose`.

## 2. Language

**English only** for code, specs, commits, PRs, issues, and all docs in this store.

Product SPA i18n: EN + JA only — do not add locales without maintainer approval.

## 3. Repository layout (sibling workspace)

| Path | Role |
|------|------|
| `migration-toolkit-sdd/` (GitHub: `rhcl-sdd`) | OpenSpec store — specs, `docs/`, `ai-specs/` |
| `../migration-toolkit-rhcl/` | Product code |

Open **`rhcl-sdd.code-workspace`** in Cursor. `/opsx-apply` edits the product repo.

**Canonical product audit**: `docs/technical-specifications.md` in this store (+ product `.cursor/rules/*.mdc`).

## 4. Linked standards (this store)

- [Backend Standards](./backend-standards.md)
- [Frontend Standards](./frontend-standards.md)
- [Conversion Architecture](./conversion-architecture.md)
- [Documentation Standards](./documentation-standards.md)
- [API Spec](./api-spec.yml)
- [Data Model](./data-model.md)
- [Development Guide](./development_guide.md)
- [SDD Backlog](./sdd-backlog.md)
- [Skills Inventory](./skills-inventory.md)
- [OpenSpec Tasks Mandatory Steps](./openspec-tasks-mandatory-steps.md)

**Product Cursor rules** (apply when editing product code): `migration-toolkit-rhcl/.cursor/rules/*.mdc`

## 5. Agent roles

| Agent | When |
|-------|------|
| `solution-architect.md` | `design.md`, #40, cross-cutting architecture |
| `backend-developer.md` | Quarkus implementation |
| `frontend-developer.md` | React/PatternFly implementation |
| `product-strategy-analyst.md` | Epic scope (optional) |

Skills: `ai-specs/skills/` — see [skills-inventory.md](./skills-inventory.md).

## 6. SDD workflow

1. `enrich-us` on GitHub issue
2. `/opsx-propose <change-name>`
3. `/opsx-apply` → `../migration-toolkit-rhcl/`
4. `/verify` — tests + `openspec validate` + verify-report
5. `/code-review` — RHCL PR-style review (Major/Moderate/Minor/Nit)
6. `/adversarial-review` — red-team (fresh session preferred)
7. `/opsx-archive`
8. `/commit` — PR on product repo

Store id: `rhcl-sdd` — `openspec list --store rhcl-sdd`

## 7. Product conventions

- CODEOWNERS review on substantive PRs (@pcastelo)
- Stacked PRs: merge root first, merge `main` into feature branch
- `NAMESPACE_PLACEHOLDER` in deploy manifests
- `VITE_API_URL` bake-time — never `REACT_APP_*`
- #169 before new bulk-fetch paths
- #170: no new positional args on `generateReadme(...)`

## 8. Post-apply discipline

Scope changes after `/opsx-apply` but before `/opsx-archive` → update OpenSpec artifacts first, then code.
