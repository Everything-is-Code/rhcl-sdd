## Why

[#228](https://github.com/Everything-is-Code/migration-toolkit-rhcl/issues/228): when the backend runs locally without an active OpenShift session (`oc login`), cluster version detection soft-fails and Compatibility Check shows a misleading **Kuadrant/RHCL operator not detected** warning—even when the target cluster has RHCL installed. The root cause is conflating **cluster unreachable** with **operator absent**: `softFailDefault` sets `kuadrantPresent: false` and `CompatibilityService` emits capability warnings as if the cluster had been probed. Workshop attendees waste time checking the cluster instead of running `oc login`.

## What Changes

- **Backend — `clusterReachable` flag:** extend `ClusterCapabilities` (surfaced on `GET /api/cluster/versions`) so `softFailDefault` sets `clusterReachable: false` and successful detect sets `clusterReachable: true`. Profile override (`source: profile`) is unchanged (user-assumed matrix; no live probe).
- **Backend — compatibility gating:** when `clusterReachable` is false, `CompatibilityService` MUST NOT emit capability-tagged warnings that require a live cluster probe (`kuadrantPresent`, `ossmMatchesOcp`, capability-tagged CORS fallback). Instead emit a single **Cluster connection** WARNING with actionable guidance (`oc login` for local dev; in-cluster deployment note).
- **Backend — genuine gaps unchanged:** when `clusterReachable: true` and Kuadrant is actually absent, existing Kuadrant warning behavior is preserved.
- **Frontend — Connection page UX:** distinguish **OpenShift cluster reachability** (backend kubeconfig) from **3scale connection** (Admin API token). `ClusterVersionsPanel` shows a prominent warning banner when unreachable, semantic copy for Kuadrant/RHCL (unreachable vs absent vs detected), and labels fallback version values as defaults—not detected.
- **Frontend — decouple cluster panel visibility:** show `ClusterVersionsPanel` even before 3scale connect so local devs can diagnose `oc login` without completing the 3scale form first.
- **i18n:** new/changed strings in `en.json` and `ja.json`.
- **Tests:** `ClusterVersionServiceTest`, `CompatibilityServiceTest`, `clusterCapabilityUi` unit tests, `ClusterVersionsPanel` smoke tests.

## Capabilities

### New Capabilities

- `cluster-detection`: cluster reachability contract, compatibility warning gating, and Connection-page presentation rules for detected vs unreachable vs profile-assumed versions.

### Modified Capabilities

_(none)_

## Impact

- **Backend DTOs:** `ClusterCapabilities.java`, `ClusterVersionsResponse` consumers.
- **Backend services:** `ClusterVersionService` (`softFailDefault`, `detectOrDefault`), `CompatibilityService` (`checkCapabilityWarnings`, optional CORS policy gate).
- **Backend controllers:** `ExportController`, `ConversionController` pass reachability into compatibility (via capabilities flag; no new REST params).
- **Frontend:** `api/types.ts`, `ClusterVersionsPanel.tsx`, `ConnectionPage.tsx`, `utils/clusterCapabilityUi.ts`, `locales/en.json`, `locales/ja.json`.
- **APIs:** additive field on existing `ClusterVersionsResponse`; no breaking HTTP contract.
- **Out of scope:** in-app OpenShift login/kubeconfig upload (#228 Option B), replacing Fabric8 client, changing conversion YAML when Kuadrant is genuinely missing.
