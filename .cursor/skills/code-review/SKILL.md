---
name: code-review
description: AI code review for RHCL PRs — Major/Moderate/Minor/Nit findings, English, no auto-approve. Use after /verify, before /commit.
author: RHCL / adapted from LIDR SpecBoot + code-auditing
version: 1.1.0
---
# code-review Skill

Independent review of changes for the active OpenSpec change or current branch diff.

**Preferred agent:** `quality-reviewer` — run this skill with the `quality-reviewer` agent for best results (independent perspective, no implementer bias).

## Input

`$ARGUMENTS` — change name, PR number, or "branch" for current diff.

## Steps

1. **Scope** — diff against `main` in `../migration-toolkit-rhcl/` or files listed in change `tasks.md`.

2. **Read context** — change artifacts, `design.md`, `docs/conversion-architecture.md` if conversion work.

3. **Review dimensions**
   - Spec compliance (vs OpenSpec spec scenarios)
   - ArchUnit / layering (if backend)
   - Security (secrets, tokens, RBAC scope)
   - #40 architecture alignment for conversion code
   - #170 — no new `generateReadme` positional args
   - #169 — bulk fetch patterns
   - Tests added/updated appropriately

4. **Output format** (RHCL convention — for PR comment or local report)

   ```
   ## AI Code Review

   _Disclaimer: AI-generated review. Human maintainer review required._

   **Change:** <name> | **Issue:** #N

   ### Major
   - ...

   ### Moderate
   - ...

   ### Minor
   - ...

   ### Nit
   - ...
   ```

5. **Save** — `openspec/changes/<name>/review-report.md` when reviewing a change.

6. **Rules**
   - English only
   - Never auto-approve unless user explicitly asks
   - Non-blocking items for closing issues → suggest consolidated follow-up issue

## Optional deep audit

For broad audits use `code-auditing` skill instead.
