# verify-fe report

| Field | Value |
|-------|-------|
| Timestamp | 2026-08-27T13:15:00-03:00 |
| Mode | full |
| Outcome | **PASS** |

## Static tests

| Check | Result |
|-------|--------|
| `npm run typecheck` | PASS |
| `npm test` (vitest, 72 tests) | PASS |

## E2E (Playwright)

| Check | Result |
|-------|--------|
| `npm run test:e2e` | PASS (6/6, ~53s) |

### Products exercised

- `rhcl_seed_claim_role_chain`
- `rhcl_seed_claim_cache_chain`
- `rhcl_seed_keycloak_roles`
- `rhcl_seed_auth_chain`
- `rhcl_seed_anonymous`
- Regression: switching API updates httproute metadata (#229)

## Environment

| Prerequisite | Status |
|--------------|--------|
| Backend `http://localhost:8080/q/health/ready` | 200 |
| Frontend `http://localhost:5173/` (curl) | 200 |
| 3scale lab | `https://3scale-admin.apps.cluster-4vhhf.dyn.redhatworkshops.io` |

## Credentials

- used: yes
- stored: no (session env only)
- token: `****a3a03` (masked)

## E2E helper fixes applied this session

- Read YAML from `tabpanel` (table preview), not hidden `pre`
- Tab selection uses `exact: true` (`policy.yaml` vs `authorizationpolicy.yaml`)
- API list: paginate pages (products 21–23 are on page 2); do not use client-side search alone
- Wait for `Fetching APIs from 3scale...` to clear before selecting a product
- Vitest excludes `e2e/` directory

## Failures

None.
