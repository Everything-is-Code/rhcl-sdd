---
name: verify-fe
description: Frontend verification with live 3scale Playwright E2E (YAML tab checks). Prompts for lab URL and token in chat before running. Use after /opsx-apply when UI or conversion wizard changed, or before /verify on frontend-heavy changes.
author: RHCL
version: 1.1.0
---
# verify-fe Skill

Run frontend unit tests plus **live** Playwright E2E against the migration wizard and seed YAML expectations.

**Separate from `/verify`:** this skill always handles 3scale lab credentials interactively. Never commit tokens or paste them into repo files.

## Input

`$ARGUMENTS` — optional:

- OpenSpec change name (for report path only)
- Comma-separated `rhcl_seed_*` keys to run (default: all in `yaml-expectations.ts`)
- `quick` — skip Playwright, only `typecheck` + `npm test`

## Prerequisites (agent checks)

1. **Product repo** `../migration-toolkit-rhcl/frontend/`
2. **Backend** responding: `GET http://localhost:8080/q/health/ready` → 200
3. **Frontend** dev server on port **5173** (Vite often binds `[::1]:5173` on Windows)

**Windows health-check trap:** `Invoke-WebRequest http://localhost:5173/` often returns **404** even when the app works in the browser. **Do not** tell the user the frontend is down based on that alone.

Use one of these instead:

```powershell
# Preferred on Windows
curl.exe -s -o NUL -w "%{http_code}" http://localhost:5173/
# Expect 200

# Fallback: index.html probe
curl.exe -s -o NUL -w "%{http_code}" http://localhost:5173/index.html
```

Or check the port is listening: `netstat -ano | findstr :5173`

If the user confirms the app loads in the browser, **trust that** and proceed — Playwright is the real E2E readiness check.

4. If backend is down, start it (or ask the user). Only ask to restart frontend when **both** curl and browser fail.

## Step 1 — Obtain lab credentials (required unless `quick`)

**Do not proceed with Playwright until the user provides credentials in chat.**

Ask explicitly:

> Para `/verify-fe` necesito credenciales del lab 3scale (solo para esta sesión, no se commitean):
> 1. **Admin URL** (ej. `https://3scale-admin.apps....`)
> 2. **Personal Access Token**

Accept only in the user message. **Never**:

- Write tokens to `e2e/.env.local`, `.env`, or git-tracked files
- Echo the full token back in the final summary (mask: `****` + last 4 chars)
- Store in OpenSpec artifacts or `verify-report.md`

Set **shell env vars for the current command only** (inline on the same `npm run test:e2e` invocation or `$env:VAR = '...'` in PowerShell for that session step).

Optional overrides (ask only if user mentions them):

- `E2E_BASE_URL` (default `http://localhost:5173`)
- `VITE_API_URL` (default `http://localhost:8080`)

## Step 2 — Frontend static checks (always)

```bash
cd ../migration-toolkit-rhcl/frontend
npm run typecheck
npm test
```

Any failure → **FAIL** (stop before E2E).

## Step 3 — Playwright setup (unless `quick`)

```bash
cd ../migration-toolkit-rhcl/frontend
npm install
npx playwright install chromium
```

## Step 4 — Playwright E2E + YAML verification (unless `quick`)

Run with credentials in env (example bash):

```bash
cd ../migration-toolkit-rhcl/frontend
E2E_SKIP_WEBSERVER=true \
THREESCALE_ADMIN_URL='<from user>' \
THREESCALE_ACCESS_TOKEN='<from user>' \
npm run test:e2e
```

PowerShell example:

```powershell
cd ../migration-toolkit-rhcl/frontend
$env:E2E_SKIP_WEBSERVER = "true"
$env:THREESCALE_ADMIN_URL = "<from user>"
$env:THREESCALE_ACCESS_TOKEN = "<from user>"
npm run test:e2e
```

**What E2E validates:** `e2e/migration-workflow.spec.ts` drives the wizard and `e2e/yaml-expectations.ts` asserts each YAML tab (`policy.yaml`, `httproute.yaml`, `authorizationpolicy.yaml`, etc.) per seed product.

To run a subset:

```bash
npm run test:e2e -- -g "claim_cache_chain"
```

## Step 5 — Report

Write **`verify-fe-report.md`**:

- If `$ARGUMENTS` includes a change name: `openspec/changes/<name>/verify-fe-report.md`
- Else: `openspec/changes/_scratch/verify-fe-report.md` (create `_scratch/` if needed)

Include:

| Section | Content |
|---------|---------|
| Timestamp | ISO date |
| Mode | `full` or `quick` |
| Static tests | typecheck + vitest pass/fail |
| E2E | pass/fail/skipped |
| Products exercised | keys from `yaml-expectations.ts` or subset |
| Failures | Playwright trace path if any |
| Credentials | `used: yes, stored: no (session env only)` — **never** the token value |

## Outcome

- **PASS** — typecheck + vitest green; E2E green (or `quick` mode)
- **FAIL** — any red test; list failing product key + YAML file from assertion message
- **BLOCKED** — user did not provide credentials when E2E was required → ask again; do not mark PASS

## When to use

| Situation | Command |
|-----------|---------|
| OpenSpec change + backend only | `/verify` |
| UI, wizard, YAML viewer, or seed E2E | `/verify-fe` (this skill) |
| Before archive on frontend work | `/verify` then `/verify-fe` |

## Rules

- English for reports and skill artifacts; may ask credentials in Spanish if user prefers
- New `rhcl_seed_*` case → extend `frontend/e2e/yaml-expectations.ts` in the same change
- Link failures to GitHub issues when known (e.g. #229 stale YAML metadata)
