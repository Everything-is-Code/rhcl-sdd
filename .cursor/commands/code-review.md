---
name: "/code-review"
id: "code-review"
category: "Workflow"
description: "AI code review (Major/Moderate/Minor/Nit) for active change or branch"
---

Review code changes for the active OpenSpec change or current branch.

**Input:** Change name, PR `#N`, or omit for current branch diff in `../migration-toolkit-rhcl/`.

**Action:** Load and follow the **`code-review`** skill (`ai-specs/skills/code-review/SKILL.md`).

**Output:** Review comment block + optional `review-report.md` in change folder.

**Rules:** English, ranked findings, no auto-approve unless user asks.

**Next:** `/commit` or `/opsx-archive` if verify already passed.
