# Documentation Standards

## Language

English only for agent-facing and maintainer docs in this store and the product repo.

## OpenSpec Artifacts

Each change under `openspec/changes/<name>/` should include (per `spec-driven` schema):

- `proposal.md` — why, scope, non-goals
- `spec.md` or `specs/` — requirements and scenarios
- `design.md` — technical approach
- `tasks.md` — checkbox tasks sized for one session

Follow [openspec-tasks-mandatory-steps.md](./openspec-tasks-mandatory-steps.md).

## Product README

Major feature or API changes must update `migration-toolkit-rhcl/README.md` sections: API Reference, Features, Data Model as needed.

## Archive

On `/opsx-archive`, optionally add a summary under `migration-toolkit-rhcl/docs/archive/` if maintainers want git-visible traceability (pattern from legacy ApiShift archives).

## Cross-Links

- GitHub issue ↔ OpenSpec change name (e.g. `policy-cors-conversion` ↔ #149 child issue)
- PR description links change name and issue number
