## Why

The `service/conversion/` package contains 15 shared support/helper classes used by generators and contributors, with **zero direct tests**. These classes are only exercised transitively through `ConversionServiceTest`, leaving significant logic untested in isolation. Adding dedicated unit tests for this package is the next coverage step after the controller tier (PR-2) and unblocks PR-4 (generators) and PR-5 (contributors) which depend on stable, well-tested support infrastructure. This is **PR-3** of issue [#210](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/210) (raise backend Codecov baseline from ~37% to tiered targets).

## What Changes

- **New tests**: Dedicated JUnit 5 unit tests for 15 source files in `service/conversion/`: AuthPolicySupport, BackendResolver, BackendType, ContributorOrdering, ConversionContext, ConversionYamlSupport, HttpRouteSupport, JwtClaimCheckSupport, PolicyConfigSupport, RateLimitSupport, ReadmeNotes, ReadmeSupport, RegistryDiscoveryMarkers, ResolvedBackend, SecretSupport.
- **No production code changes** — test-only.

## Capabilities

### New Capabilities

_None — this change adds tests only, no new behavioral capabilities._

### Modified Capabilities

_None — no spec-level behavior changes. `skip_specs: true` is set in `.openspec.yaml` because this is a pure test/quality change._

## Impact

- **Code**: `backend/src/test/java/com/redhat/migrationtoolkit/rhcl/service/conversion/` — new test files
- **CI**: JaCoCo coverage for `service/conversion/` rises from ~0% to ≥50%; Codecov `codecov/project/backend` baseline rises after merge
- **Dependencies**: None — tests use plain JUnit 5 + Mockito (no `@QuarkusTest`)
- **APIs**: No changes
- **Related issues**: Part of [#210](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/210) (PR-3); unblocks PR-4 (generators) and PR-5 (contributors). Does not affect #169, #149, or #198
