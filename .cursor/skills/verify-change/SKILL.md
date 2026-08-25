---
name: verify-change
description: Validate implementation against OpenSpec change artifacts and run tests. Use after /opsx-apply, before /code-review or /opsx-archive.
author: RHCL / adapted from LIDR SpecBoot
version: 1.0.0
---
# verify-change Skill

Validate that product code matches the active OpenSpec change before archive or PR.

## Input

`$ARGUMENTS` — change name (e.g. `conversion-strategy-registry`) or GitHub issue `#40`.

## Steps

1. **Resolve change** — infer from args/context or `openspec list --store rhcl-sdd --json`.

2. **Read artifacts** — from `openspec/changes/<name>/`: proposal, spec(s), design, tasks.

3. **OpenSpec validate**
   ```bash
   openspec validate <change-name> --type change --store rhcl-sdd --strict
   openspec status --change <change-name> --json --store rhcl-sdd
   ```
   All tasks in `tasks.md` must be `- [x]` unless explicitly deferred in artifacts.

4. **Run tests** (product repo)
   ```bash
   cd ../migration-toolkit-rhcl/backend && mvn test
   cd ../migration-toolkit-rhcl/frontend && npm run typecheck && npm test
   ```
   Optional for large backend changes: `mvn verify`.

5. **Spec compliance** — for each requirement/scenario in spec artifacts:
   - Trace to code change or test
   - Note gaps as blocking or follow-up

6. **Write verify report** — append to change folder:
   `openspec/changes/<name>/verify-report.md` with:
   - Change name + issue link
   - Test results (pass/fail, note Windows CRLF if applicable)
   - `openspec validate` result
   - Scenario checklist (pass/fail/partial)
   - Blocking issues vs non-blocking follow-ups

7. **Outcome**
   - **PASS** — all tasks done, tests green (or CI-equivalent), no blocking spec gaps → suggest `/code-review` then `/opsx-archive`
   - **FAIL** — list blockers; do not archive

## Rules

- English only
- Windows: `ConversionServiceTest` CRLF failures — check CI before blocking
- Do not mark PASS if tasks.md has open checkboxes for in-scope work
