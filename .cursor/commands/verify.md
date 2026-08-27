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

**Tests run:** `mvn test`, `npm run typecheck`, `npm test`, `openspec validate`.

**Frontend E2E:** Not run here — use **`/verify-fe`** (asks for 3scale URL + token in chat). `/verify` checks for `verify-fe-report.md` when `frontend/` changed.

**Output:** `verify-report.md` in the change folder + PASS/FAIL summary.

**Next:** `/verify-fe` (if UI) → `/code-review` → `/opsx-archive` → `/commit`
