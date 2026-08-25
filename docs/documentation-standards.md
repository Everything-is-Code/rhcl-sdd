# Documentation Standards

## Language

English only for agent-facing and maintainer docs.

## Source-of-truth hierarchy

| Layer | Location | Role |
|-------|----------|------|
| **Product audit** | `docs/technical-specifications.md` (this store) | Ground truth from code audit |
| **Product Cursor rules** | `migration-toolkit-rhcl/.cursor/rules/*.mdc` | File-scoped rules when editing product code |
| **SDD store docs** | `rhcl-sdd/docs/` | Agent-oriented summaries + OpenSpec context |
| **OpenSpec changes** | `openspec/changes/<name>/` | Per-change proposal, spec, design, tasks |

SDD docs must stay **consistent** with TECHNICAL_SPECIFICATIONS.md. If they diverge, fix SDD docs or file a doc-sync task.

## OpenSpec artifacts

Per `spec-driven` schema:

- `proposal.md` — why, scope, non-goals
- `spec.md` / `specs/` — requirements and scenarios
- `design.md` — technical approach (**solution-architect**)
- `tasks.md` — checkbox tasks ([mandatory steps](./openspec-tasks-mandatory-steps.md))

## Product README

Feature/API changes → update `migration-toolkit-rhcl/README.md` (API Reference, Features, Data Model).

Major pattern changes → also update `TECHNICAL_SPECIFICATIONS.md`.

## Archive

Optional summary in `migration-toolkit-rhcl/docs/archive/` on `/opsx-archive` (legacy ApiShift pattern).

## Cross-links

- GitHub issue ↔ OpenSpec change name
- PR description: issue + change name + `Closes #N`

## Do not duplicate symlink entry points

`AGENTS.md`, `CLAUDE.md`, `codex.md`, `GEMINI.md` are **separate files** — not symlinks to `base-standards.md`. Rules live in `docs/base-standards.md`.
