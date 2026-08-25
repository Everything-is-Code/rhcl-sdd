---
name: "/adversarial-review"
id: "adversarial-review"
category: "Workflow"
description: "Red-team review before archive (independent session preferred)"
---

Run independent adversarial review before `/opsx-archive`.

**Input:** Change name, `#issue`, or PR URL.

**Action:** Load **`adversarial-review`** skill (`ai-specs/skills/adversarial-review/SKILL.md`).

**When:** After `/verify` passes; prefer a **different agent session** than `/opsx-apply`.

**Output:** `adversarial-review.md` in change folder + PASS/FAIL verdict.

**Next:** `/opsx-archive` only on PASS or PASS WITH GAPS (no blockers/majors).
