## Context

See [proposal.md](./proposal.md). Today `ClusterVersionService.softFailDefault()` sets `source: default`, applies OCP/GAPI fallback versions, and builds `ClusterCapabilities` with `kuadrantPresent: false`. `CompatibilityService.checkCapabilityWarnings()` treats that as a real cluster gap. The Connection page gates `ClusterVersionsPanel` on 3scale `connected`, conflating Admin API login with OpenShift kubeconfig reachability.

Relevant code (product repo):

- `ClusterVersionService` — `softFailDefault`, `detectOrDefault`, `capabilitiesFrom`
- `CompatibilityService` — `checkCapabilityWarnings`, CORS capability branch in `checkPolicies`
- `ClusterVersionsPanel`, `ConnectionPage`, `clusterCapabilityUi.ts`

## Goals / Non-Goals

**Goals:**

- Single source of truth: `capabilities.clusterReachable` drives compatibility gating and Connection UI state.
- Eliminate false Kuadrant warnings when failure mode is unreachable/unauthorized.
- Connection page clearly separates: (1) cluster reachable?, (2) resolution source (detected / default / profile), (3) version values with correct semantics.
- Preserve conversion behavior: soft-fail still supplies conservative defaults for `corsNative` / `retriesSupported` via existing fallback versions.

**Non-Goals:**

- In-app OpenShift credential storage (#228 Option B).
- Full `detectStatus` enum refactor (#228 Option C) beyond `clusterReachable`.
- Frontend-only filtering of compatibility items (no duplicate gating logic in React).
- Changing YAML output when Kuadrant is genuinely missing on a reachable cluster.

## Decisions

### 1. Add `clusterReachable` on `ClusterCapabilities` (not parallel `source` param on `check()`)

**Choice:** boolean on existing DTO, set in `ClusterVersionService` only.

| Set by | `clusterReachable` |
|--------|-------------------|
| `detectOrDefault` success (`source: detected`) | `true` |
| `softFailDefault` (`source: default`) | `false` |
| `applyProfile` (`source: profile`) | omit or leave default; compatibility uses existing profile assumptions |

**Rationale:** Flows through existing `resolveCapabilities()` in `ExportController` / `ConversionController` without signature changes. Avoids coupling `CompatibilityService` to `ClusterVersionsResponse.source`.

**Alternative rejected:** Pass `source` string into `check()` — duplicates semantics and drifts from profile behavior.

### 2. Compatibility gating in one place (`CompatibilityService`)

When `!capabilities.clusterReachable`:

- Skip `checkCapabilityWarnings` entries (`kuadrantPresent`, `ossmMatchesOcp`).
- Skip capability-tagged CORS warning in `checkPolicies` (the branch with `capability: corsNative`).
- Add `checkClusterConnectionWarning()` → single item:
  - `name`: `"Cluster connection"`
  - `status`: `WARNING`
  - `capability`: `clusterReachable` (new id, mirrors existing pattern)
  - Message: actionable, no secrets/paths.

When `clusterReachable` is true, behavior unchanged.

**Alternative rejected:** `capabilities = null` on soft-fail — breaks conversion opts (`corsNative`, `retriesSupported`).

### 3. Connection UI state machine (`clusterCapabilityUi.ts`)

Pure helper (vitest-covered):

```text
getClusterConnectionUiState(versions) →
  'reachable' | 'unreachable' | 'profile'
```

Derived from `capabilities.clusterReachable` + `source`:

- `source === 'profile'` → `profile` (banner: assumed versions)
- `clusterReachable === false` → `unreachable` (warning banner + Kuadrant unreachable copy)
- else → `reachable` (success/meta + live values)

Kuadrant cell renderer:

| State | Display |
|-------|---------|
| reachable + version | version string |
| reachable + no version | "Operator not found on cluster" |
| unreachable | "Connect to cluster to detect RHCL/Kuadrant" |
| profile 4.21 | assumed-present helper text |

Fallback OCP/GAPI: append muted "(fallback default)" when `source === 'default'`.

### 4. Decouple cluster panel from 3scale connect

**Choice:** Change `shouldShowClusterVersionsCard` to always return `true` (or remove gate); load versions on `ConnectionPage` mount regardless of 3scale state.

**Rationale:** Workshop debugging — user sees `oc login` issue before filling 3scale form.

`ConnectionPage` `useEffect`: call `loadVersions(false)` on mount unconditionally (not only when `appState.connection.connected`).

### 5. API contract

Additive field only:

```json
"capabilities": {
  "clusterReachable": false,
  "corsNative": false,
  ...
}
```

Update `frontend/src/api/types.ts` `ClusterCapabilities` interface. No OpenAPI file change required unless `docs/api-spec.yml` documents cluster versions response — add field if present.

### 6. Tests (TDD order per base-standards)

**Backend:**

- `ClusterVersionServiceTest`: soft-fail → `clusterReachable false`; detected → `true`
- `CompatibilityServiceTest`:
  - unreachable + JWT service → cluster connection warning, no `kuadrantPresent`
  - reachable + no kuadrant → existing kuadrant warning
  - unreachable + cors policy → no `corsNative` capability warning

**Frontend:**

- `clusterCapabilityUi.test.ts`: state helper + Kuadrant display key selection
- `ClusterVersionsPanel.test.tsx` (jsdom smoke): unreachable shows warning variant

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| Double warning (cluster connection + other items) | Gate only probe-dependent capability warnings; policy-list warnings unchanged |
| Profile `auto` + unreachable still shows defaults | Label as fallback; cluster connection compatibility item explains |
| Conversion uses conservative defaults when unreachable | Documented existing behavior; unchanged |
| Extra API calls on Connection mount | Cached `ClusterVersionService`; single GET on page load |

## Migration Plan

1. Ship backend + frontend in one PR (additive API field; old clients ignore `clusterReachable`).
2. No DB migration.
3. Rollback: revert PR; field ignored by older frontend builds.

## Open Questions

_(none — scope locked to A2 + Connection UX per exploration)_
