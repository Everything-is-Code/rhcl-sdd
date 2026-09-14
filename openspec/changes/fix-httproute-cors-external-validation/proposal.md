# Proposal: Fix HTTPRoute CORS + external URLRewrite duplication and validation gap

**Issue:** [#332](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/332)  
**Trigger:** Functional migration of `rhcl_seed_cors` on OCP 4.21 workshop cluster

## Problem

`rhcl_seed_cors` with external backend (`httpbin.org`) and `corsNative=true` emits `httproute.yaml` with **duplicate `filters:` keys** per rule (URLRewrite + native CORS). Package validation reports **valid** although output is not acceptable for apply.

## Goals

1. Single `filters` array per HTTPRoute rule combining URLRewrite (external host) and native CORS.
2. `ValidationService` fails closed on duplicate YAML keys and basic HTTPRoute structural defects.
3. Regression test matching workshop repro (external + corsNative).

## Non-goals (this change)

- Full OpenAPI / CRD schema validation against cluster
- Runtime CORS parity on cluster (#286–#288)
- Gateway TLS secret auto-generation (separate follow-up if needed)

## Success criteria

- `ConversionServiceTest` reproduces workshop scenario; passes after fix
- `ValidationServiceTest` rejects duplicate-key httproute sample
- Manual re-run of `rhcl_seed_cors` export validates clean in UI
