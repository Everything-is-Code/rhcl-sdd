---
name: verify-change
description: Validate implementation against OpenSpec change artifacts and run tests. Use after /opsx-apply, before /code-review or /opsx-archive. For live UI/YAML E2E use /verify-fe instead.
author: RHCL / adapted from LIDR SpecBoot
version: 1.2.0
---
# verify-change Skill

Validate that product code matches the active OpenSpec change before archive or PR.

**Live Playwright / 3scale lab:** use **`/verify-fe`** — it asks for URL + token in chat. This skill does **not** run E2E or handle credentials.

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

4. **Detect product-repo scope** (in `../migration-toolkit-rhcl/`)
   ```bash
   git diff --name-only origin/main...HEAD
   ```
   Set `FRONTEND_TOUCHED` if any path under `frontend/`.

5. **Run tests** (product repo)
   ```bash
   cd ../migration-toolkit-rhcl/backend && mvn test
   cd ../migration-toolkit-rhcl/frontend && npm run typecheck && npm test
   ```
   Optional for large backend changes: `mvn verify` (exclude `PlaywrightE2EIT` if needed: `mvn -Dtest='!PlaywrightE2EIT' verify`).

6. **Frontend E2E gate** — if `FRONTEND_TOUCHED`:
   - Check for `openspec/changes/<name>/verify-fe-report.md` with outcome **PASS** (same day or same branch).
   - If missing → recommend **`/verify-fe`** before archive; note in verify-report as **follow-up required** (blocking for UI-heavy changes unless user explicitly defers).

7. **Spec compliance** — for each requirement/scenario in spec artifacts:
   - Trace to code change or test
   - Conversion/seed: `ConversionServiceTest` and/or `frontend/e2e/yaml-expectations.ts`

8. **Write verify report** — `openspec/changes/<name>/verify-report.md`:
   - Change name + issue link
   - `FRONTEND_TOUCHED` / `BACKEND_TOUCHED`
   - `mvn test`, `npm test` results
   - Link to `verify-fe-report.md` if present
   - `openspec validate` result
   - Scenario checklist
   - Blocking vs non-blocking follow-ups

9. **Outcome**
   - **PASS** — tasks done, unit tests green, spec gaps none, `verify-fe` PASS when frontend touched (or deferred with user OK)
   - **FAIL** — blockers listed; do not archive

## Rules

- English only
- Windows: `ConversionServiceTest` CRLF — check CI before blocking
- Do not mark PASS if tasks.md has open checkboxes for in-scope work
- Never ask for or store 3scale tokens in this skill — use `/verify-fe`
