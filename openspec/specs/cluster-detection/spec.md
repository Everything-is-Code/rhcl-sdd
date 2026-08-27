# cluster-detection Specification

## Purpose
Defines how the toolkit detects OpenShift cluster versions from the backend, exposes reachability to clients, and gates compatibility warnings so unreachable clusters are not reported as missing operators.

## Requirements

### Requirement: Cluster versions response exposes reachability

The `GET /api/cluster/versions` response SHALL include a boolean `clusterReachable` field on `capabilities` indicating whether the backend successfully probed the OpenShift cluster (not merely applied fallback defaults).

#### Scenario: Successful live detection

- **WHEN** the backend resolves cluster versions with `source: detected` after a successful probe
- **THEN** `capabilities.clusterReachable` is `true`

#### Scenario: Soft-fail default after auth or probe failure

- **WHEN** cluster detection falls back to `source: default` because the Kubernetes client is unavailable, unauthorized, or OCP version cannot be read
- **THEN** `capabilities.clusterReachable` is `false`
- **AND** `source` is `default`

#### Scenario: Manual profile override

- **WHEN** the user selects a manual cluster profile (`source: profile`)
- **THEN** `capabilities.clusterReachable` is not required to be `true` (profile-assumed matrix; live probe skipped)
- **AND** existing profile capability assumptions remain unchanged

### Requirement: Compatibility check does not claim missing operators when cluster is unreachable

When `capabilities.clusterReachable` is `false`, the compatibility check SHALL NOT emit warnings that assert cluster capability gaps discovered only by live probe.

#### Scenario: Unreachable cluster with auth-requiring service

- **WHEN** compatibility check runs for a service that needs Kuadrant (e.g. JWT auth or rate-limit policies)
- **AND** `capabilities.clusterReachable` is `false`
- **THEN** the result MUST NOT include a `kuadrantPresent` capability warning
- **AND** the result MUST NOT include an `ossmMatchesOcp` capability warning
- **AND** the result MUST NOT include a capability-tagged CORS native-missing warning

#### Scenario: Unreachable cluster emits cluster connection warning

- **WHEN** compatibility check runs
- **AND** `capabilities.clusterReachable` is `false`
- **THEN** the result includes exactly one **Cluster connection** item with status `WARNING`
- **AND** the message guides the user to authenticate the backend to OpenShift (`oc login` for local dev) or deploy in-cluster, then refresh cluster versions

#### Scenario: Reachable cluster with missing Kuadrant

- **WHEN** compatibility check runs for a Kuadrant-dependent service
- **AND** `capabilities.clusterReachable` is `true`
- **AND** `capabilities.kuadrantPresent` is `false`
- **THEN** the existing Kuadrant/RHCL operator-not-detected warning is emitted
- **AND** no **Cluster connection** warning is emitted solely for reachability

#### Scenario: Reachable cluster with Kuadrant present

- **WHEN** compatibility check runs for a Kuadrant-dependent service
- **AND** `capabilities.clusterReachable` is `true`
- **AND** `capabilities.kuadrantPresent` is `true`
- **THEN** no Kuadrant capability warning is emitted

### Requirement: Connection page distinguishes cluster reachability from 3scale connection

The Connection page SHALL present OpenShift cluster status independently of the 3scale Admin API connection form.

#### Scenario: Cluster panel visible without 3scale connect

- **WHEN** the user opens the Connection page without an active 3scale connection
- **THEN** the cluster versions panel is visible and can be refreshed

#### Scenario: Unreachable cluster banner

- **WHEN** `capabilities.clusterReachable` is `false`
- **THEN** the cluster versions panel shows a prominent warning (not info-level) explaining the backend cannot reach OpenShift
- **AND** provides actionable `oc login` / in-cluster deployment guidance

#### Scenario: Kuadrant display when unreachable

- **WHEN** `capabilities.clusterReachable` is `false`
- **THEN** Kuadrant/RHCL version MUST NOT be shown as a bare em dash implying operator absence
- **AND** copy indicates detection requires cluster login (or equivalent unreachable state text)

#### Scenario: Kuadrant display when reachable and absent

- **WHEN** `capabilities.clusterReachable` is `true`
- **AND** no Kuadrant version was detected
- **THEN** the UI indicates the operator was not found on the probed cluster

#### Scenario: Fallback version values labeled

- **WHEN** `source` is `default`
- **THEN** OCP and Gateway API values shown are labeled as fallback defaults, not live-detected versions

#### Scenario: Profile override labeling

- **WHEN** `source` is `profile`
- **THEN** the UI indicates versions are assumed from the selected profile, not live-detected

### Requirement: Localization covers new cluster states

All new or changed user-visible strings for cluster reachability, connection warnings, and Kuadrant display states SHALL exist in both `en.json` and `ja.json`.

#### Scenario: English and Japanese parity

- **WHEN** the Connection page or compatibility results display cluster-reachability messaging
- **THEN** corresponding i18n keys exist in `en.json` and `ja.json`
- **AND** `locales.smoke.test.ts` continues to pass
