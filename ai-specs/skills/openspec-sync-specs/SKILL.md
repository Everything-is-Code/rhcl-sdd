---
name: openspec-sync-specs
description: Sync delta specs from a change to main specs without archiving. Use when updating main specs from change deltas.
license: MIT
compatibility: Requires openspec CLI.
metadata:
  author: openspec
  version: "1.0"
  rhcl-store: rhcl-sdd
---

Sync delta specs from a change to main specs (agent-driven intelligent merge).

**Store:** `rhcl-sdd` — use `--store rhcl-sdd` on all `openspec` commands.

**Input:** Change name (prompt if ambiguous via `openspec list --store rhcl-sdd --json`).

## Steps

1. Find delta specs: `openspec/changes/<name>/specs/*/spec.md`
2. For each capability delta, read main spec at `openspec/specs/<capability>/spec.md`
3. Apply **ADDED** / **MODIFIED** / **REMOVED** / **RENAMED** sections intelligently (partial scenario merges OK)
4. Create main spec if capability is new
5. Summarize what changed per capability

## Delta format

```markdown
## ADDED Requirements
### Requirement: ...
#### Scenario: ...
- **WHEN** ...
- **THEN** ...

## MODIFIED Requirements
## REMOVED Requirements
## RENAMED Requirements
- FROM: `### Requirement: Old`
- TO: `### Requirement: New`
```

## Guardrails

- Read both delta and main before editing
- Preserve content not mentioned in delta
- Idempotent merges
- Change stays active — archive only after `/verify` + adversarial review

## Success output

```
## Specs Synced: <change-name>

Updated main specs:
**<capability>**: Added/Modified/Removed requirements...

Change remains active until archive.
```
