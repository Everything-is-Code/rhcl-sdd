---
name: sync-agent-symlinks
description: Sync .cursor/skills and .claude/skills with ai-specs/skills after add/remove/rename. Canonical source is ai-specs/skills.
author: LIDR.co / RHCL adapted
version: 1.0.0
---
# sync-agent-symlinks Skill

Keep agent mirrors aligned with `ai-specs/skills/` as canonical source.

## Targets

- `.cursor/skills/`
- `.claude/skills/`

## Windows note

Symlinks may require Developer Mode or admin. If `New-Item -ItemType SymbolicLink` fails, **copy** `SKILL.md` (and skill folder contents) into mirror paths and report copy-vs-symlink in the sync report.

## Workflow

1. **Inventory** canonical skills (`ai-specs/skills/*/SKILL.md`) vs mirror entries
2. **Classify** mirrors: `linked` | `broken` | `orphan` | `conflict` | `external`
3. **Plan:** to_add, to_fix, to_remove, to_skip
4. **Apply:** add/fix canonical links; remove orphan canonical symlinks only
5. **Never** auto-delete non-symlink real directories without explicit user request

## Report

- Canonical count
- Per mirror: added, fixed, removed, conflicts, skipped
- Remaining blockers

## Reference (Unix)

```bash
ln -s ../../ai-specs/skills/<name> .cursor/skills/<name>
```

## RHCL

Also sync `.cursor/agents/` from `ai-specs/agents/` when new agent roles are added (copy on Windows if symlink fails).
