---
name: "/verify"
id: "verify"
category: "Workflow"
description: "Verify implementation against OpenSpec change + run tests (post-apply)"
---

Validate implementation against the active OpenSpec change.

**Store:** `rhcl-sdd` — use `--store rhcl-sdd` on OpenSpec CLI commands.

**Input:** Change name (e.g. `/verify conversion-strategy-registry`) or issue `#40`.

**Action:** Load and follow the **`verify-change`** skill (`ai-specs/skills/verify-change/SKILL.md`).

**Output:** `verify-report.md` in the change folder + PASS/FAIL summary.

**Next:** `/code-review` → `/opsx-archive` → `/commit`
