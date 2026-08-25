---
name: show-spec-working
description: Live demo of a spec/feature — browser walkthrough or curl API proof. Use when user says "show me", "demo", "walk me through", "how X works".
author: LIDR.co / RHCL adapted
version: 1.0.0
---
# show-spec-working Skill

Demonstrate a spec or change in a **runnable** way — not a summary report.

## Trigger phrases

`show me X`, `demo X`, `walk me through X`, `show X working`, `how X works`, `prove X works`

## Inputs

- GitHub `#N`, change name, endpoint, or frontend route
- Infer from active OpenSpec change or session context

## Anti-report guardrail

Never finish with only analysis. If blocked, state the blocker and what is needed to continue the live demo.

## Modality

| Type | When |
|------|------|
| `frontend` | UI in spec |
| `backend-only` | API only |
| `mixed` | Both |

## Frontend path

1. Start services if needed:
   ```bash
   cd ../migration-toolkit-rhcl/backend && mvn quarkus:dev
   cd ../migration-toolkit-rhcl/frontend && VITE_API_URL=http://localhost:8080 npm run dev
   ```
2. Browser MCP: navigate app (default `http://localhost:5173`)
3. Walk through acceptance scenarios step by step; verify visible results
4. Keep browser open unless user asks to close

## Backend path

1. Identify endpoints from spec or `docs/api-spec.yml`
2. Run explicit `curl` commands with masked tokens
3. If CREATE/UPDATE/DELETE: restore state after demo
4. Report key response evidence

## Completion

```markdown
Spec demo completed for: <change>

Frontend:
- <step / result>

Backend:
- <curl + response note>

Data restore: <restored / not needed / failed>

Next: continue in browser or ask to close.
```

## RHCL notes

- No 3scale token in chat; use env or placeholder
- Cluster apply demos need live `oc` context — confirm with user before apply
