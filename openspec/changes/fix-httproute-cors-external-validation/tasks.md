# Tasks: fix-httproute-cors-external-validation (#332)

## Prerequisites

- Issue: [#332](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/332)
- SDD change: `fix-httproute-cors-external-validation`
- Product branch: `feature/332-httproute-cors-external-validation` from `main`
- Repro artifact: `rhcl-seed-cors/` functional export (workshop, `rhcl_seed_cors`, ns `fmeneses-rhcl-test`)

Ready for Apply: **Yes**

## Phase 1: RED tests

- [ ] 1.1 Branch `feature/332-httproute-cors-external-validation` from `main`
- [ ] 1.2 `ConversionServiceTest`: external httpbin + cors + `corsNative=true` — assert one `filters:` per rule, URLRewrite + CORS present
- [ ] 1.3 `ValidationServiceTest`: feed defective `httproute.yaml` from repro — expect `valid=false`
- [ ] 1.4 `HttpRouteBuilderTest`: assert `filters:` key count ≤ 1 per rule after inject

## Phase 2: GREEN conversion fix

- [ ] 2.1 Fix `HttpRouteBuilder.injectCorsIntoRule` — merge CORS into existing filters array
- [ ] 2.2 Verify `mvn -Dtest=HttpRouteBuilderTest,ConversionServiceTest test` green

## Phase 3: Validation hardening

- [ ] 3.1 `ValidationService`: duplicate YAML key detection (ERROR)
- [ ] 3.2 Optional: HTTPRoute warning when multiple `filters:` keys detected in raw content
- [ ] 3.3 `mvn -Dtest=ValidationServiceTest test` green

## Phase 4: Minor / follow-up (same PR or split)

- [ ] 4.1 ServiceEntry double `---` separator (if generator bug confirmed)
- [ ] 4.2 Gateway TLS `certificateRefs` — validation WARNING when secret missing from package

## Phase 5: Verify + PR

- [ ] 5.1 Re-run functional `rhcl_seed_cors` on workshop cluster; attach clean `httproute.yaml` snippet to #332
- [ ] 5.2 `cd backend && mvn verify` (advisory Playwright skip if no cluster)
- [ ] 5.3 PR `Closes #332`
