# test-coverage-cluster-compat

## Why

[#222](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/222) slices 1–2 shipped in PRs [#224](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/224) / [#225](https://github.com/Everything-is-Code/migration-toolkit-rhcl/pull/225). OpenSpec `test-coverage-service-data-layers` archived with slices 3–4 deferred.

Remaining ROI: extend unit tests for `ClusterVersionService`, `CompatibilityService`, and thin `ConversionService` orchestration paths not hit by contributor/generator tests.

## What Changes

Test-only work in up to **two PR slices** (same change, sequential merge optional):

1. **Cluster + compat** — `ClusterVersionServiceTest`, `CompatibilityServiceTest` gaps (`ossmCompatibilityTable`, `sanitize`, `capabilitiesFrom` retries, logging `json_object_config` warning, score levels).
2. **ConversionService** — overload/orchestration paths (`loggingTarget`/`anonymousTarget` string overloads, `includeMigratedFromLabel` stripping).

No production code unless a tiny testability seam is unavoidable.

## Capabilities

None — `skip_specs: true`.

## Impact

- **Code**: `backend/src/test/java/**` only.
- **CI**: Codecov `backend` ratchet; closes remaining #222 test backlog after merge.
