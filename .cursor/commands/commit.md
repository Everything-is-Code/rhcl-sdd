---
name: "/commit"
id: "commit"
category: "Workflow"
description: "Create focused commit and PR after verify + code-review"
---

Create a focused commit and Pull Request for the product repo.

**Input:** Optional issue/change id (e.g. `/commit #40`) or empty for all scoped changes.

**Action:** Load and follow the **`commit`** skill (`ai-specs/skills/commit/SKILL.md`).

**Target repo:** `../migration-toolkit-rhcl/` (not the SDD store unless docs-only SDD commit requested).

**Prerequisites:** `/verify` should PASS for the change; `/code-review` completed.

**Notes:**
- Use `gh pr create` with issue link (`Closes #N`)
- Conventional Commits: `feat:`, `fix:`, `refactor:`, `test:`, `docs:`
- Never commit secrets
