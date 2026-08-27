---
name: "/verify-fe"
id: "verify-fe"
category: "Workflow"
description: "Frontend verify — typecheck, vitest, live Playwright YAML E2E (asks for 3scale URL + token in chat)"
---

Frontend verification with **live 3scale** Playwright E2E.

**Store:** `rhcl-sdd`

**Input:** Optional change name, `quick` (unit tests only), or product filter (e.g. `claim_cache_chain`).

**Action:** Load and follow **`verify-fe`** skill (`ai-specs/skills/verify-fe/SKILL.md`).

**Credentials:** The agent **must ask you** for Admin URL and PAT in chat before running E2E. Credentials are used only in the shell session — never committed.

**Tests:**
1. `npm run typecheck` + `npm test`
2. `npm run test:e2e` — wizard + YAML tabs vs `yaml-expectations.ts`

**Prerequisites:** Backend `:8080` and frontend `:5173` running. On Windows, use `curl.exe` to probe the frontend — `Invoke-WebRequest` often falsely reports 404 while the browser works.

**Output:** `verify-fe-report.md` in the change folder (or `openspec/changes/_scratch/`).

**Workflow:** `/opsx-apply` → `/verify-fe` (if frontend) → `/verify` → `/code-review` → `/opsx-archive`
